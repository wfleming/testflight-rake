# TestFlight Rake utility
# Will Fleming <will@jwock.org> 2011
#
# A collection of rake tasks for an iOS project to make building & deploying
# builds to TestFlight much easier.
#
# Setup:
# 1. Drop this Rakefile into your project root.
# 2. Ensure you have the httpclient gem installed
# 3. Currently we depend on you versioning your builds using agvtool. Do that.
# 4. Create a file called .rakeenv in the same directory. Add configuration here.
#    The following is pretty much the minimum of what you need to put here:
#
#    .rakenv contents:
#    set(:app_name, YOUR_APP_NAME)
#    set(:environment, 'TestFlight') # you can use other environment tasks later if you prefer, but you need a default
#    set(:xcode_configuration, CONFIGURATION_TO_BUILD)
#    set(:testflight_team_token, YOUR_TEAM_TOKEN)
#    set(:testflight_api_token, YOUR_API_TOKEN)
#    set(:testflight_distribution_lists, TESTFLIGHT_DIST_LISTS_TO_SEND_TO)  # not strictly required, but emails won't be sent if you don't
#
# Use:
# `rake release` is all you should really need. Look around if you want more.
#
# Maybe I'll put some more detail on configuration, options, flow, and potential
# improvements later. But not right now.

require 'fileutils'
require 'rubygems'
require 'httpclient'

# utility functions
def set(key, value)
  ENV[key.to_s.upcase] = value
end

def fetch(key)
  ENV[key.to_s.upcase]
end


# Load up some default settings, then user environment
set(:no_tag, false)
set(:edit_release_notes, true)
if File.exists? ('.rakeenv')
  load '.rakeenv'
else
  puts "WARNING: no .rakeenv found. If you didn't set everything in your environment, you will be unhappy."
end


### TASKS ###

task :require_environment do
  env = fetch(:environment)
  if env.nil? ||  0 == env.length
    # TODO - do more checking here? or delete entirely?
    raise Exception.new("No environment specified. You need to specify an environment")
  end
end

desc "bump the current release number"
task :bump_version do
  puts "=== Bumping the build number"
  puts `agvtool bump -all`
  build_number = `agvtool vers -terse`.chomp
  puts `git commit *-Info.plist */*-Info.plist *.xcodeproj/project.pbxproj -m "Bumping version number to #{build_number}"`
  puts `git push`
  puts
end

desc 'Tag and push tag for current release'
task :tag_release => [:require_environment] do
  if fetch(:no_tag).nil?
    puts "=== Tagging the current build"
    env = fetch(:environment)

    # tag a specific tag so we can refer to this version
    build_number = `agvtool vers -terse`.chomp
    date_str = Time.now.strftime('%Y-%m-%d')
    puts `git tag -fam '' #{env}-#{date_str}-build-#{build_number} HEAD`
    puts `git push -f origin #{env}-#{date_str}-build-#{build_number}`

    # move current -> previous
    current  = `git show-ref --tags --hash --abbrev #{env}-current`.chomp
    if current && current.length > 0
      puts `git tag -fam '' #{env}-previous #{env}-current`
      puts `git push -f origin refs/tags/#{env}-previous`
    end

    # re-tag current
    puts `git tag -fam '' #{env}-current HEAD`
    puts `git push -f origin refs/tags/#{env}-current`

    puts
  end
end

desc 'Make a best guess at what the release notes for a release should be.'
task :release_notes => [:require_environment] do
  #TODO - don't think this goes well if this is your first release (i.e. no -previous tag)
  #TODO - also try to be smart & abandon ship if release notes for this version already exist
  env = fetch(:environment)
  previous_tag = "#{env}-previous"
  current_tag = "#{env}-current"
  log = `git log --pretty="* %s [%an, %h]" #{previous_tag}...#{current_tag}`
  file_name = "/tmp/#{fetch(:app_name)}-rake-release-notes-#{`agvtool vers -terse`.chomp}.txt"
  FileUtils.rm(file_name, :force => true) if File.exists?(file_name)
  File.open(file_name, 'w') { |f| f.write(log) }
  set(:release_notes_path, file_name)
  unless fetch(:skip_showing_release_notes)
    `$EDITOR #{file_name}`
  end
end

desc 'Compile the .app'
task :compile_app => [:require_environment] do
  conf = fetch(:xcode_configuration)
  app_name = fetch(:app_name)

  puts "=== Compiling the app"
  puts "* running xcodebuild..."
  xcode_out = `xcodebuild -sdk iphoneos -configuration #{conf}`
  if 0 != $?.exitstatus
    puts "* ERROR - xcodebuild failed"
    puts "* XCode Output:\n\n#{xcode_out}\n\nEND XCode Output\n\n"
    raise Exception.new("xcodebuild failed")
  else
    puts "* xcodebuild completed."
  end
end

desc 'Create a signed .ipa from the .app'
task :sign_ipa => [:require_environment] do
  conf = fetch(:xcode_configuration)
  app_name = fetch(:app_name)

  puts "=== Building the .ipa"
  app_path = "build/#{conf}-iphoneos/#{app_name}.app"
  ipa_path = "build/#{conf}-iphoneos/#{app_name}.ipa"

  if !File.exists?(app_path)
    raise Exception.new("#{app_path} not found. Did you forgot to run compile_app?")
  end

  set(:ipa_path, ipa_path)
  FileUtils.rm_f(ipa_path) if File.exists?(ipa_path)
  puts "* Going to run: `xcrun -sdk iphoneos PackageApplication -v \"#{app_path}\" -o \"#{ipa_path}\" --sign \"Mint Digital\" --embed \"Speedo_AdHoc.mobileprovision\"`" #DEBUG
  xcrun_output = `xcrun -sdk iphoneos PackageApplication -v "#{app_path}" -o "#{ipa_path}" --sign "Mint Digital" --embed "Speedo_AdHoc.mobileprovision"`
  puts "NB: this command will claim it failed, saying it was 'unable to create' the ipa. Don't worry, everything's fine unless you see errors after this."
  # xcrun is expected to fail because it's dumb...but it mostly gets there.
  # it just fails on the zip step because it's looking in the wrong place.
  # so we figure out the right path from the output and go after it.
  pattern = /(\/var\/.*\/Payload\/#{app_name}\.app): explicit requirement satisfied/
  match = pattern.match(xcrun_output)
  app_bundle_path = match[1]

  if !File.exists?(app_bundle_path)
    puts "* ERROR: couldn't find the signed .app - thought it was #{app_bundle_path}"
    raise Exception.new("Couldn't sign app bundle")
  else
    cwd = FileUtils.pwd
    FileUtils.cd("#{app_bundle_path}/../..")
    `/usr/bin/zip --symlinks --verbose --recurse-paths "#{cwd}/#{ipa_path}" ./Payload`
    FileUtils.cd(cwd)
  end

  if !File.exists?(ipa_path)
    puts "* ERROR: the ipa doesn't exist where it should"
    raise Exception.new("Couldn't sign the app bundle")
  end
end

desc 'Build the project & sign the .ipa'
task :build_release => [:compile_app, :sign_ipa]

desc 'upload an already-built IPA path to testflight'
task :upload_to_testflight do
  set(:skip_showing_release_notes, 'true')
  Rake::Task["release_notes"].invoke

  puts "=== Posting ipa to testflight"

  ipa_path = fetch(:ipa_path)
  release_notes_path = fetch(:release_notes_path)

  if fetch(:edit_release_notes)
    # TODO - insert #comment lines in release notes informing user they can edit,
    # then strip them afterwards before uploading
    `$EDITOR #{release_notes_path}`
  end

  if ipa_path.nil? || !File.exists?(ipa_path)
    raise Exception.new("No .ipa was found! ipa_path is '#{ipa_path}'")
  end

  testflight_endpoint = 'http://testflightapp.com/api/builds.json'
  notify_teammates = true

  # HTTPClient
  release_notes_string = File.open(release_notes_path, 'r').read
  response = HTTPClient.post(testflight_endpoint, {
    :file => File.new(ipa_path),
    :notes => release_notes_string,
    :api_token => fetch(:testflight_api_token),
    :team_token => fetch(:testflight_api_token),
    :notify => notify_teammates,
    :distribution_lists => fetch(:testflight_distribution_lists)
  })
  if (response.status != 200)
   puts "* ERROR posting file: got a #{response.status} response:"
  else
    puts "* SUCCESS:"
  end
  puts response.body.content
end

desc 'Upload a built release to TestFlight'
task :distribute_release => [:build_release, :upload_to_testflight]

desc 'Tag, build, & release.'
task :release => [:bump_version, :tag_release, :distribute_release]