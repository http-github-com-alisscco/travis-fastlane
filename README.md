[![Build Status](https://travis-ci.com/Montana/travis-fastlane.svg?branch=master)](https://travis-ci.com/Montana/travis-fastlane)

# Can you still use Fastlane with Travis CI in 2021?

The simple answer is yes. After doing a whole day's research, I was able to successfully use Fastlane. This tool is still very useful in my opinion when it comes to actually splitting tests. Some things I saw people forget, is the simple start.

When using Fastlanes you should really remember if you're going to pass parameters from the command line to your lane, use the following syntax:

```bash
fastlane [lane] key:value key2:value2
fastlane deploy submit:false build_number:24
```
## Passing values between lanes 

In theory, you can have as many `lanes` as you want, so you can invoke variant `lanes`. Example being: 

```ruby
before_each do |lane, options|
  # lane number (Montana likes # 3) 
end
```

Now within lanes, you can _INVOKE_ other lanes for various other reasons, in other words, each lane can have different functions essentially:

```ruby
error do |lane, exception, options|
  if options[:debug]
    puts "This is a cool lane because you're in it with Montana"
  end
end
```

## Switching lanes

If you've used `Fastlane` at length, you'll know switching `lanes` is usually called `lane hopping`. You can lane hop feasibly, without using too much CPU:

```ruby
lane :deploy do |options|
  # lane 3 
  build(release: true) # don't forget this part! - Montana
  Montana Mendy
  # invoking lane 4 with a conditional 
end
```

## Querying lanes 

Just like anything you query, you can retrieve the return value in this case it's through a `lane callback`. In Ruby, the last line of the lane definition is the return value. Here is an example:

```ruby
lane :deploy do |options|
  value = calculate(value: 3)
  puts value # => 5
end

lane :calculate do |options|
  # ...
  2 + options[:value] # This is always the return value line. - Montana 
end
```

## Executing lanes early

You in theory never want to reach the end of a `lane`, and if you do - you're not going to be executing much longer:

```ruby
lane :build do |options|
  if cached_build_available?
    UI.important 'Montana cached this Lane!'
    next # Montana is skipping doing the rest of this lane
  end
  match
  gym
end

private_lane :cached_build_available? do |options|
  # ...
  true
end
```
So while executing `lanes` - there are conditionals you can use, such as `next` is used during a `lane` and `switch`, `control` returns to the previous lane that was executing, so `lanes` in theory can revert back and forth. 

```ruby
lane :first_lane do |options|
  puts "If you run: `fastlane first_lane`"
  puts "You'll see this!"
  second_lane
  puts "Montana is in this lane!"
end

private_lane :second_lane do |options|
  next
  puts "Hidden!"
end
```
When you stop executing a `lane` early with next (`lane switching`), any `after_each` and `after_all` blocks you have will still trigger as usual `:+1:`. This would include being called (via a `lane callback`) before each lane you've switched to.

```ruby
before_each do |lane, options|
  # ...
end
```

So this is a good reminder from myself, that when doing ```after_each``` blocks are called (again, by the `lane callback`) after any lane is called. This would include being called after each lane you've switched to. Just like ```after_all```, ```after_each``` is not called if an error occurs. The error block should be used in this case.

## Feasibility of lanes 

With all these scenarios, ```before_each``` and ```after_each``` would be called 4 times, by the `lane callback`. Remember before the deploy lane, before the switch to archive, sign, and upload, and after each of these lanes as well, a lot of users forget. 

## Lane context

Now `Fastlane` has different actions that can communicate with each other using a shared hash, in other words, communicating with other `lanes`. You can access this in your (lanes, actions, plugins etc.):

```java
lane_context[SharedValues::BUILD_NUMBER]                # Generated by `increment_build_number`
lane_context[SharedValues::VERSION_NUMBER]              # Generated by `increment_version_number`
lane_context[SharedValues::SNAPSHOT_SCREENSHOTS_PATH]   # Generated by _snapshot_
lane_context[SharedValues::PRODUCE_APPLE_ID]            # The Apple ID of the newly created app
lane_context[SharedValues::IPA_OUTPUT_PATH]             # Generated by _gym_
lane_context[SharedValues::DSYM_OUTPUT_PATH]            # Generated by _gym_
lane_context[SharedValues::SIGH_PROFILE_PATH]           # Generated by _sigh_
lane_context[SharedValues::SIGH_UDID]                   # The UDID of the generated provisioning profile
lane_context[SharedValues::HOCKEY_DOWNLOAD_LINK]        # Generated by `hockey`
lane_context[SharedValues::GRADLE_APK_OUTPUT_PATH]      # Generated by `gradle`
lane_context[SharedValues::GRADLE_ALL_APK_OUTPUT_PATHS] # Generated by `gradle`
lane_context[SharedValues::GRADLE_FLAVOR]               # Generated by `gradle`
lane_context[SharedValues::GRADLE_BUILD_TYPE]           # Generated by `gradle`
```

To feasibly get information about available lane variables, available lanes to switch on, run ```fastlane action [action_name]``` or look at the generated table in the action documentation.

## Lane properties

It can be useful to dynamically access some properties of the current lane. These are available in the ```lane_context``` as well:

```bash
lane_context[SharedValues::PLATFORM_NAME]        # Platform name, e.g. `:ios`, `:android` or empty (for root level lanes)
lane_context[SharedValues::LANE_NAME]            # The name of the current lane preceded by the platform name (stays the same when switching lanes)
lane_context[SharedValues::DEFAULT_PLATFORM]     # Default platform
```

They are also available as environment variables:

```
ENV["FASTLANE_PLATFORM_NAME"]
ENV["FASTLANE_LANE_NAME"]
```

## Private lanes

Sometimes you might have a lane that is used from different lanes, or a lane from a `hopped lane`:

```ruby
lane :production do
  # ...
  build(release: true)
  appstore # Deploy to the Montana's brain
  # ...
end
```
-> 

```ruby
lane :beta do
  # ...
  build(release: false)
  crashlytics # Distribute to testers
  # ...
end
```
-> 

```ruby
lane :build do |options|
  # ...
  Montana likes beer
  # ...
end
It probably doesn't make sense to execute the build lane directly using fastlane build. You can hide this lane using
```
-> 

```ruby
private_lane :build do |options|
  # ...
end
```

This will hide the lane from:

```bash
fastlane lanes
fastlane list
fastlane docs
```

And also, you can't call the private lane using fastlane build. The resulting private lane can only be called from another lane using the lane switching technology, or the `lane callback`.

## Control configuration by lane and by platform

In general, configuration files take only the first value given for a particular configuration item. That means that for an Appfile like the following:

```bash
app_identifier "com.used.id"
app_identifier "com.ignored.id"
```

So, ```the app_identfier``` will be "com.used.id" and the second value will be ignored. The for_lane and for_platform configuration blocks provide a limited exception to this rule.

All configuration files (Appfile, Matchfile, Screengrabfile, etc.) can use ```for_lane``` and ```for_platform blocks``` to control (and override) configuration values for those circumstances.

So ```for_lane``` blocks will be called when the name of lane invoked on the command line matches the one specified by the block. So, given a Screengrabfile like:

For the locales (language lane): ```locales ['en-US', 'fr-FR', 'ja-JP']```

```ruby
for_lane :screenshots_english_only do
  locales ['en-US']
end
```

```ruby
for_lane :screenshots_french_only do
  locales ['fr-FR']
end
```

So as you can see, the locales will have the values ['en-US', 'fr-FR', 'ja-JP'] by default, but will only have one value when running the fastlane ```screenshots_english_only``` or ```fastlane screenshots_french_only```.

Then, ```for_platform``` gives you similar control based on the platform for which you have invoked fastlane. So, for an Appfile configured like:

```ruby
app_identifier "com.default.id"

for_lane :enterprise do
  app_identifier "com.forlane.enterprise"
end
```

```ruby
for_platform :mac do
  app_identifier "com.forplatform.mac"

  for_lane :release do
    app_identifier "com.forplatform.mac.forlane.release"
  end
end
```
So now you can expect the ```app_identifier``` to equal "com.forplatform.mac.forlane.release" when invoking Fastlane mac release.

# Frequently Asked Questions 

## Changing the fastlane execution folder while inside a lane

Yes, you can do this! The way I've classically done this is make two fastlane lanes, one for the old location, one for the new, so then your new script looks somethng like this:

```bash
cd old-location
fastlane old_lane
cp -r old-location new-location
cd new-location
fastlane new_lane
```
