# TestFlight Rake utility
# Will Fleming <will@jwock.org> 2011
# https://github.com/wfleming/testflight-rake
#
# A collection of rake tasks for an iOS project to make building & deploying
# builds to TestFlight much easier.

require 'fileutils'
require 'rubygems'
require 'httpclient'

# utility functions
def set(key, value)
  if false == value
    value = 'FALSE'
  elsif true == value
    value = 'TRUE'
  end
  ENV[key.to_s.upcase] = value
end

def fetch(key)
  val = ENV[key.to_s.upcase]
  if 'FALSE' == val
    val = false
  elsif 'TRUE' == val
    val = true
  end
  val
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
  # again, auto-pushing is annoying. add a flag?
  #puts `git push`
  puts
end

desc 'Tag and push tag for current release'
task :tag_release => [:require_environment] do
  if fetch(:no_tag).nil? || !fetch(:no_tag)
    puts "=== Tagging the current build"
    env = fetch(:environment)

    # tag a specific tag so we can refer to this version
    # This generates a *lot* of tags, but some people might find it useful.
    # should add a flag & turn it off by default
    # build_number = `agvtool vers -terse`.chomp
    # date_str = Time.now.strftime('%Y-%m-%d')
    # puts `git tag -fam '' #{env}-#{date_str}-build-#{build_number} HEAD`
    # puts `git push -f origin #{env}-#{date_str}-build-#{build_number}`

    # move current -> previous
    current  = `git show-ref --tags --hash --abbrev #{env}-current`.chomp
    head = `git show-ref --hash --abbrev HEAD`.chomp
    
    if current == head
      puts "* Looks like this release was already tagged."
      return
    end
    
    if current && current.length > 0
      puts `git tag -fam '' #{env}-previous #{env}-current`
      # don't push by default - again, maybe add a flag?
      #puts `git push -f origin refs/tags/#{env}-previous`
    else # there was no current - assume this is first build, tag first commit as -previous
      first_commit = `git log --pretty="%H" | tail -1`.chomp
      puts `git tag -fam '' #{env}-previous #{first_commit}`
    end

    # re-tag current
    puts `git tag -fam '' #{env}-current HEAD`
    # don't push by default - again, maybe add a flag?
    #puts `git push -f origin refs/tags/#{env}-current`

    puts
  end
end

desc 'Make a best guess at what the release notes for a release should be.'
task :release_notes => [:require_environment] do
  env = fetch(:environment)
  
  file_name = "/tmp/#{fetch(:app_name)}-#{env}-release-notes-#{`agvtool vers -terse`.chomp}.txt"
  set(:release_notes_path, file_name)
  
  if File.exists?(file_name)
    puts "* It looks like release notes already exist for this version"
  else  
    previous_tag = "#{env}-previous"
    current_tag = "#{env}-current"
  
    # this will not go well if these tags don't exist.
    if 0 == `git tag -l #{previous_tag}`.length || 0 == `git tag -l #{current_tag}`.length
      puts "* ERROR: abandoning release notes, couldn't find tags"
      return
    end

    log = `git log --pretty="* %s [%an, %h]" #{previous_tag}...#{current_tag}`  
    File.open(file_name, 'w') { |f| f.write(log) }
  end
  
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
  codesign_id = fetch(:codesign_identity)
  provisioning_profile = fetch(:provisioning_profile_path)
  puts "* Going to run: `xcrun -sdk iphoneos PackageApplication -v \"#{app_path}\" -o \"#{ipa_path}\" --sign \"#{codesign_id}\" --embed \"#{provisioning_profile}\"`" #DEBUG
  xcrun_output = `xcrun -sdk iphoneos PackageApplication -v "#{app_path}" -o "#{ipa_path}" --sign "#{codesign_id}" --embed "#{provisioning_profile}"`
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
    :team_token => fetch(:testflight_team_token),
    :notify => notify_teammates,
    :distribution_lists => fetch(:testflight_distribution_lists)
  })
  if (response.status != 200)
   puts "* ERROR posting file: got a #{response.status} response:"
  else
    puts "* SUCCESS:"
  end
  puts response.body
end

desc 'Upload a built release to TestFlight'
task :distribute_release => [:build_release, :upload_to_testflight]

desc 'Tag, build, & release.'
task :release => [:bump_version, :tag_release, :distribute_release]