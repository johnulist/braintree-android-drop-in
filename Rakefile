require 'rake'

TMP_CHANGELOG_FILE = "/tmp/braintree-android-drop-in-release.md"

task :default => :tests

desc "Run Android lint on all modules"
task :lint do
  sh "./gradlew clean lint"
end

desc "Run all Android tests"
task :tests => :lint do
  output = `adb devices`
  if output.match(/device$/)
    begin
      log_listener_pid = fork { exec 'ruby', 'script/log_listener.rb' }
      sh "./gradlew --continue test connectedAndroidTest"
    ensure
      `kill -9 #{log_listener_pid}`
    end
  else
    puts "Please connect a device or start an emulator and try again"
    exit 1
  end
end

desc "Publish current version as a SNAPSHOT"
task :publish_snapshot => :tests do
  abort("Version must contain '-SNAPSHOT'!") unless get_current_version.end_with?('-SNAPSHOT')

  sh "./gradlew clean :Drop-In:uploadArchives"
end

desc "Interactive release to publish new version"
task :release do
  puts "What version are you releasing? (x.x.x format)"
  version = $stdin.gets.chomp

  prompt_for_change_log(version)
  increment_version_code
  update_version(version)

  sh "./gradlew clean :Drop-In:uploadArchives"
  sh "./gradlew :Drop-In:closeRepository"
  puts "Sleeping for one minute to allow Drop-In modules to close"
  sleep 60
  sh "./gradlew :Drop-In:promoteRepository"

  post_release(version)
end

def prompt_for_change_log(version)
  last_version = `git tag | tail -1`.chomp
  tmp_change_log = "#{version}"
  tmp_change_log += "\n\n# Please enter a summary of the changes below."
  tmp_change_log += "\n# Lines starting with '# ' will be ignored."
  tmp_change_log += "\n#"
  tmp_change_log += "\n# Changes since #{last_version}:"
  tmp_change_log += "\n#"
  tmp_change_log += "\n# "
  tmp_change_log += `git log --pretty=format:"%h %ad%x20%s%x20%x28%an%x29" --date=short #{last_version}..`.gsub("\n", "\n# ")
  tmp_change_log += "\n#"
  tmp_change_log += "\n"
  File.foreach("CHANGELOG.md") do |line|
    tmp_change_log += "# #{line}"
  end
  IO.write(TMP_CHANGELOG_FILE, tmp_change_log)

  puts "\n"
  sh "$EDITOR #{TMP_CHANGELOG_FILE}"

  new_changes = ""
  File.foreach(TMP_CHANGELOG_FILE) do |line|
    if !line.start_with?("# ") && !line.start_with?("#\n")
      new_changes += line
    end
  end

  IO.write("CHANGELOG.md",
    File.open("CHANGELOG.md") do |file|
      file.read.gsub("# Braintree Android Drop-In Release Notes\n", "# Braintree Android Drop-In Release Notes\n\n## #{new_changes.chomp}")
    end
  )
end

def post_release(version)
  puts "\nArchives are uploaded! Committing and tagging #{version} and preparing for the next development iteration"
  sh "git commit -am 'Release #{version}'"
  sh "git tag -aF #{TMP_CHANGELOG_FILE} #{version}"

  update_version("#{version}-SNAPSHOT")
  sh "git commit -am 'Prepare for development'"

  puts "\nDone. Commits and tags have been created. If everything appears to be in order, hit ENTER to push."
  $stdin.gets

  sh "git push origin master #{version}"

  puts "\nSend release notification email to braintree-sdk-announce@googlegroups.com. Press ENTER when done."
  $stdin.gets
end

def get_current_version
  current_version = nil
  File.foreach("Drop-In/build.gradle") do |line|
    if match = line.match(/versionName '(\d+\.\d+\.\d+(-SNAPSHOT)?)'/)
      current_version = match.captures
    end
  end

  return current_version[0]
end

def increment_version_code
  new_build_file = ""
  File.foreach("Drop-In/build.gradle") do |line|
    if line.match(/versionCode (\d+)/)
      new_build_file += line.gsub(/versionCode \d+/, "versionCode #{$1.to_i + 1}")
    else
      new_build_file += line
    end
  end
  IO.write('Drop-In/build.gradle', new_build_file)
end

def update_version(version)
  IO.write("Drop-In/build.gradle",
    File.open("Drop-In/build.gradle") do |file|
      file.read.gsub(/versionName '\d+\.\d+\.\d+(-SNAPSHOT)?'/, "versionName '#{version}'")
    end
  )
end
