---
layout: post
title: "Automating XCode 7 UI Tests with Charles and apimocker"
date: 2015-08-16 20:28:30 +0200
comments: true
categories: [XCode 7, UI Tests, CI]
---

Xcode 7 brings automated UI testing on board. It doesn't seem perfectly stable yet, nor does it support all of the UI elements (as of beta 4), but it makes adding UI tests to the app fairly easy. Instead of setting up the environment for Appium, Calabash, or other frameworks, we only have to tap a checkbox to have a project with UI tests target set up.

<!-- more -->

To make UI tests self contained we can use HTTP mocking library (OHHTTPStub for a good example) and stub all the requests. This is pretty good solution already, but what always bothered me is that the app needs to modified for that, and the more we modify the app the farther away it goes from the binary we will are going to release.

The other solution is to use mock server outside of the app. This lets our binary send HTTP requests like it would in production environment and all the magic happens outside of our binary. I set this up using [apimocker](https://www.npmjs.com/package/apimocker) together with [Charles HTTP Proxy](http://www.charlesproxy.com). Apimocker is simple to set up API mocking tool (it also sports many useful features). Charles Proxy is actually the only tool I found so far that is capable of mapping requests.

##The test
For simplicity I will only write one test that will check for the number of forecast days. In [Rain Shield](http://github.com/yomajkel/Rain_Shield) app each day is represented by table cell, so to test if correct days number is displayed I only need to count table cells.

To quickly test for above condition, following line will suffice
```swift Asserting 7 table cells
XCTAssert(XCUIApplication().tables.elementBoundByIndex(0).cells.count == 7)
```

At the start Rain Shield makes asynchronous request and only updates it's views after receiving response. If the line above will execute immediately the test will fail. Here is where expectations come in handy. I expect for at least one cell to be present in order to execute the rest of the test.
```swift Expect table cell to be present
func cellExistsExpectation() -> XCTestExpectation {
    let cell = XCUIApplication().tables.childrenMatchingType(.Cell).elementBoundByIndex(0)
    let existPredicate = NSPredicate(format: "exists == 1")
    return expectationForPredicate(existPredicate, evaluatedWithObject: cell, handler: nil)
}
```
 
The complete test class looks like this
```swift Test for 7 weather cells to be present
class Rain_ShieldUITests: XCTestCase {
        
    override func setUp() {
        super.setUp()
        
        continueAfterFailure = false
        
        XCUIApplication().launch()

    }
    
    func test_shouldDisplay7CellsWithWeatherInformation() {
        cellExistsExpectation()
        waitForExpectationsWithTimeout(5.0, handler: nil)
        
        XCTAssert(XCUIApplication().tables.elementBoundByIndex(0).cells.count == 7)
    }
    
    func cellExistsExpectation() -> XCTestExpectation {
        let cell = XCUIApplication().tables.childrenMatchingType(.Cell).elementBoundByIndex(0)
        let existPredicate = NSPredicate(format: "exists == 1")
        return expectationForPredicate(existPredicate, evaluatedWithObject: cell, handler: nil)
    }
}
```


##Hands on request-response manipulation
Rain Shield app executes one request to [Open Weather Map API](http://api.openweathermap.org/) that gets the weather for following days and this is the request we want mapped and the response mocked.

###apimocker setup
I add `apimocker_config.json` to my project to tell apimocker what to do
```json
{
  "quiet": false,
  "port": "1337",
  "latency": 50,
  "logRequestHeaders": false,
  "webServices": {
    "openweathermap/data/2.5/forecast/daily": {
      "verbs": ["get"],
      "mockFile": "daily_forecast_london.json"
    }
  }
}
```

This tells apimocker to return content of daily_forecast_london.json file for every GET request to `localhost:1337/openweathermap/data/2.5/forecast/daily`.

###Charles setup

We need to tell Charles to map requests going to `api.openweathermap.org` onto address configured in apimocker - `localhost:1337/openweathermap`. We do that in [```Tools -> Map Remote```](http://www.charlesproxy.com/documentation/tools/map-remote/) tool. 

{% img center /images/edit_map_remote.png 'Charles Map Request config' %}

In 'Map From' section we say that host is `api.openweathermap.org` and we use wildcard `*` as path (which will map everything). 
In 'Map To' host is `localhost`, port `1337` and path is `/openweathermap/`. 

For this to work in Simulator we also need to enable Mac Proxying in Charles. Just make sure that `Proxy -> Mac OS X Proxy` is checked.

##Test Driving our setup
We only need to run apimocker now. After typing 
```sh
apimocker --config apimocker_config.json
``` 
we should see some feedback about our setup. After running Rain Shield app, Charles should show us request to localhost instead of openweather API.

{% img center /images/charles_localhost_log.png 'Charles localhost log' %}

apimocker should also print out that it mocked forecast response with `daily_forecast_london.json` content.

{% img center /images/apimocker_log.png 'apimocker log' %}

Yay! One advantage we get already is that the request is fast and is available offline, so it might be useful for developing as well.

##Automating it for CI testing
###Preparing Charles config

Charles has a nice config file that steers everything. We get the path to default config when we run Charles from command line 
```sh
/Applications/Charles.app/Contents/MacOS/Charles -headless
```

When we change configuration of Charles via UI the changes we make are reflected in this file. I reccomend first setting up our Remote Mapping and copy the file to our project once we have it. 

To run Mac OS X Proxy at startup we need to set `useHTTP` to `true`.
```xml
<macOSXConfiguration>
	<useHTTP>true</useHTTP>
	<useSOCKS>false</useSOCKS>
	<enableAtStartup>false</enableAtStartup>
</macOSXConfiguration>
```

I also don't want Charles to check for updates during testing, so we can turn it off.
```xml
<checkUpdates>false</checkUpdates>
```

We can also enable web access, so that we can configure some things via browser under `http://http://control.charles/`
```xml
<remoteControlConfiguration>
	<enabled>true</enabled>
	<allowAnonymous>true</allowAnonymous>
	<users/>
</remoteControlConfiguration>
```

And last but not least we can also see our remote mapping configuration
```xml
<entry>
	<string>Map Remote</string>
	<map>
		<toolEnabled>true</toolEnabled>
		<mappings>
			<mapMapping>
				<sourceLocation>
					<protocol>http</protocol>
					<host>api.openweathermap.org</host>
					<path>*</path>
				</sourceLocation>
				<destLocation>
					<protocol>http</protocol>
					<host>localhost</host>
					<port>1337</port>
					<path>/openweathermap/</path>
				</destLocation>
				<enabled>true</enabled>
			</mapMapping>
		</mappings>
	</map>
</entry>
```

Once we have our config file prepared we can use it to run Charles without UI 
```sh
/Applications/Charles.app/Contents/MacOS/Charles -headless -config /Users/yomajkel/Projects/Rain\ Shield/Rain\ ShieldUITests/apimock/charles.config
``` 
One thing here is that it did not work for me with relative path to config file. It has to be absolute.

##Starting our environment from script

To start Chalres and apimocker from cli we can use following script
```sh Starting our mocking services
#!/bin/bash
open -ga Charles --args -headless -config /Users/mkreft/Projects/Rain\ Shield/Rain\ ShieldUITests/apimock/charles.config
apimocker --config apimocker_config.json &

```
`open -ga` opens app in background mode, passing everything after `--args` as standard arguments to the app itself

If you don't intend to have our environment running all the time, it is also easy to quit our programs within script
```sh Ending our mocking services
#!/bin/bash
pgrep -f Charles | xargs kill
pgrep -f apimocker | xargs kill
```

