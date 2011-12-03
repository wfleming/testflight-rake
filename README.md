# TestFlight Rake utility

A collection of rake tasks for an iOS project to make building & deploying builds to TestFlight much easier.

What it automates:

* incrementing your version number
* compiling & signing your app bundle
* compiling release notes (from commit messages), which you can edit before upload

## Setup:
1. Drop this `Rakefile` into your project root.
2. Ensure you have the `httpclient` gem installed
3. Currently we depend on you versioning your builds using `agvtool`. Do that.
4. Create a file called `.rakeenv` in the same directory. Add configuration to this file.
   The following is pretty much the minimum of what you need to put here:

```ruby
set(:app_name, YOUR_APP_NAME)
set(:environment, 'TestFlight') # you can use other environment tasks later if you prefer, but you need a default
set(:xcode_configuration, CONFIGURATION_TO_BUILD)
set(:codesign_identity) # The string name of your code signing certificate
set(:provisioning_profile_path, PATH_TO_PROVISIONINGPROFILE) #haven't yet found a way to do this without a direct path reference
set(:testflight_team_token, YOUR_TEAM_TOKEN)
set(:testflight_api_token, YOUR_API_TOKEN)
set(:testflight_distribution_lists, TESTFLIGHT_DIST_LISTS_TO_SEND_TO)  # not strictly required, but emails won't be sent if you don't
```

Nb. you need an explicit path reference to the `.mobileprovision` file you're using to codesign. I personally just keep a copy of my provisioning profile in the project directory and point at that, but if you want to hunt down the system path, have fun. I haven't yet figured out a way to automatically infer the correct mobile provision, unfortunately. This (& your codesigning identity should be inferable based on the XCode Configuration, but it just hasn't proved simple to do. If you figure it out, please submit a pull request!)

## Use:
`rake release` is all you should really need. Look around if you want more.


Maybe I'll put some more detail on configuration, options, flow, and potential improvements later. But not right now.
