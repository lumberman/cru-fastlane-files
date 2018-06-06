# This file contains the fastlane.tools configuration
# You can find the documentation at https://docs.fastlane.tools
#
# For a list of all available actions, check out
#
#     https://docs.fastlane.tools/actions
#

# Uncomment the line if you want fastlane to automatically update itself
# update_fastlane

default_platform(:ios)

platform :ios do

  desc "Push a new release build to the App Store"
  lane :release do |options|
    tag = ENV['TRAVIS_TAG'] || ''

    if tag =~ /.*-android/
      next
    end

    target = ENV["CRU_TARGET"]
    submit_for_review = options.key?(:submit) && options[:submit] || false
    automatic_release = options.key?(:auto_release) && options[:auto_release] || false
    include_metadata = options.key?(:include_metadata) && options[:include_metadata] || false
    
    version_number = get_version_number(
        target: target
    )

    build_number = get_build_number

    upload_to_app_store(
        username: ENV['CRU_FASTLANE_USERNAME'],
        app_identifier: ENV["CRU_APP_IDENTIFIER"],
        build_number: build_number,
        dev_portal_team_id: ENV["CRU_DEV_PORTAL_TEAM_ID"],
        skip_screenshots: true,
        skip_metadata: !include_metadata,
        app_version: version_number,
        automatic_release: automatic_release,
        submit_for_review: submit_for_review,
    )

    cru_notify_users(message: "#{target} iOS Release Build #{version_number} (#{build_number}) submitted to App Store.")

    if submit_for_review
      cru_notify_users(message: "Build has been submitted for review and will be #{automatic_release ? 'automatically' : 'manually'} released.")
    end
  end

  desc "Push a new (beta) release build to TestFlight"
  lane :beta do
    build_number = cru_set_build_number
    target = ENV["CRU_TARGET"]
    build_branch = ENV['TRAVIS_BRANCH']

    sh('git', 'checkout', build_branch)

    ipa_path = cru_build_app

    testflight(
        username: ENV['CRU_FASTLANE_USERNAME'],
        app_identifier: ENV["CRU_APP_IDENTIFIER"],
        ipa: ipa_path,
        dev_portal_team_id: ENV["CRU_DEV_PORTAL_TEAM_ID"],
        changelog: ENV["TRAVIS_COMMIT_MESSAGE"]
    )

    cru_update_commit(message: "[skip ci] Build number bump to ##{build_number}")

    push_to_git_remote

    cru_notify_users(message: "#{target} iOS Beta Build ##{build_number} released to TestFlight.")
  end

  # Localization functions

  desc "Download latest localization files from Onesky"
  lane :cru_download_localizations do
    locales = ENV["ONESKY_ENABLED_LOCALIZATIONS"].split(',')
    filename = ENV["ONESKY_FILENAME"]
    scheme = ENV['CRU_SCHEME']

    locales.each do |locale|
      begin
        onesky_download(
            public_key: ENV["ONESKY_PUBLIC_KEY"],
            secret_key: ENV["ONESKY_SECRET_KEY"],
            project_id: ENV["ONESKY_PROJECT_ID"],
            locale: locale,
            filename: filename,
            destination: "./#{scheme}/#{locale}.lproj/#{filename}"
        )
      rescue
        puts("Failed to import #{locale}")
      end
    end

    cru_commit_localization_files(filename: filename)
  end

  desc 'Commit downloaded localization files to default branch and push to remote'
  lane :cru_commit_localization_files do |options|
    filename = options[:filename]

    begin
      git_commit(path: "*/#{filename}",
                 message: "[skip ci] Adding latest localization files from Onesky")
    rescue
      puts("Failed to commit localization files.. maybe none to commit?")
    end
  end

  # Helper functions

  lane :cru_set_build_number do
    build_number = ENV["TRAVIS_BUILD_NUMBER"]

    increment_build_number(
        build_number: build_number
    )

    build_number
  end

  lane :cru_build_app do |options|
    profile_name = options[:profile_name] || ENV["CRU_APPSTORE_PROFILE_NAME"]
    type = options[:type] || 'appstore'
    export_method = options[:export_method] || 'app-store'

    if ENV['CRU_SKIP_LOCALIZATION_DOWNLOAD'].nil?
      cru_download_localizations
    end

    automatic_code_signing(
        use_automatic_signing: false,
        profile_name: profile_name
    )

    cru_fetch_certs(type: type)

    if ENV["CRU_SKIP_COCOAPODS"].nil?
      cocoapods(
          podfile: './Podfile',
          try_repo_update_on_error: true
      )
    end

    gym(
        scheme: ENV["CRU_SCHEME"],
        export_method: export_method,
        export_options: {
            provisioningProfiles: {
                ENV["CRU_APP_IDENTIFIER"] => profile_name
            }
        }
    )
  end

  lane :cru_fetch_certs do |options|
    # Travis requires a keychain to be created to store the certificates in, however
    # using this utility to create a keychain locally will really mess up local keychains
    # and is not required for a successful build
    create_keychain(
        name: ENV["MATCH_KEYCHAIN_NAME"],
        password: ENV["MATCH_PASSWORD"],
        default_keychain: true,
        unlock: true,
        timeout: 3600,
        add_to_search_list: true
    )

    match(type: options[:type],
          username: ENV['CRU_FASTLANE_USERNAME'],
          app_identifier: ENV['CRU_APP_IDENTIFIER'],
          keychain_name: ENV["MATCH_KEYCHAIN_NAME"],
          keychain_password: ENV["MATCH_PASSWORD"])

  end

  lane :cru_update_commit do |options|
    project_file_path = "#{ENV['CRU_XCODEPROJ']}/project.pbxproj"
    info_file_path = "#{ENV['CRU_SCHEME']}/Info.plist"
    git_commit(path:[project_file_path, info_file_path],
               message: options[:message])
  end

  lane :cru_notify_users do |options|
    hipchat(
        message: options[:message],
        channel: ENV["HIPCHAT_CHANNEL"],
        version: "2",
        custom_color: "green"
    )
  end
end

