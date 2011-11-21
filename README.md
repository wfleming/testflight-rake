# TestFlight Rake utility

A collection of rake tasks for an iOS project to make building & deploying builds to TestFlight much easier.

## Setup:
1. Drop this `Rakefile` into your project root.
2. Ensure you have the `httpclient` gem installed
3. Currently we depend on you versioning your builds using `agvtool`. Do that.
4. Create a file called `.rakeenv` in the same directory. Add configuration here.
   The following is pretty much the minimum of what you need to put here:

```ruby
set(:app_name, YOUR_APP_NAME)  
set(:environment, 'TestFlight') # you can use other environment tasks later if you prefer, but you need a default  
set(:xcode_configuration, CONFIGURATION_TO_BUILD)  
set(:testflight_team_token, YOUR_TEAM_TOKEN)  
set(:testflight_api_token, YOUR_API_TOKEN)  
set(:testflight_distribution_lists, TESTFLIGHT_DIST_LISTS_TO_SEND_TO)  # not strictly required, but emails won't be sent if you don't
```

## Use:
`rake release` is all you should really need. Look around if you want more.


Maybe I'll put some more detail on configuration, options, flow, and potential improvements later. But not right now.
