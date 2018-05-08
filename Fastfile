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
    profile_name = ENV["CRU_APPSTORE_PROFILE_NAME"]
    target = ENV["CRU_TARGET"]
    submit_for_review = options.key?(:submit) && options[:submit] || false
    automatic_release = options.key?(:auto_release) && options[:auto_release] || false
    version_number = get_version_number(
        target: target
    )
    
    if options.key?(:version)
      version_number = increment_version_number(
          version_number: options[:version].gsub('v','') # Automatically increment major version number
      )
    end

    build_number = cru_set_build_number

    cru_build_app

    upload_to_app_store(
        app_identifier: ENV["CRU_APP_IDENTIFIER"],
        ipa: ENV["CRU_IPA_PATH"],
        dev_portal_team_id: ENV["CRU_DEV_PORTAL_TEAM_ID"],
        skip_screenshots: true,
        skip_metadata: true,
        app_version: version_number,
        automatic_release: automatic_release,
        submit_for_review: submit_for_review,
    )

    cru_update_commit(message: "[skip ci] Version number bump to #{version_number}, Build number bump to ##{build_number}")

    cru_notify_users("#{target} iOS Release Build #{version_number} (#{build_number}) submitted to App Store.")

    if submit_for_review
      cru_notify_users("Build has been submitted for review and will be #{automatic_release ? 'automatically' : 'manually'} released.")
    end
  end

  desc "Description of what the lane does"
  desc "Push a new (beta) release build to Crashlytics"
  lane :beta do
    build_number = cru_set_build_number
    target = ENV["CRU_TARGET"]

    cru_build_app

    testflight(
        app_identifier: ENV["CRU_APP_IDENTIFIER"],
        ipa: ENV["CRU_IPA_PATH"],
        dev_portal_team_id: ENV["CRU_DEV_PORTAL_TEAM_ID"],
    )

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
  lane :cru_commit_localization_files |options| do
    filename = options[:filename]
    default_branch = ENV['CRU_DEFAULT_BRANCH']
    build_branch = ENV['TRAVIS_BRANCH'] || ENV['TRAVIS_TAG']

    begin
      puts("Switching to branch: #{default_branch} to commit localization files.")
      sh('git', 'checkout', default_branch)
      git_pull

      git_commit(path: "*/#{filename}",
                 message: "[skip ci] Adding latest localization files from Onesky")
      push_to_git_remote
    rescue
      puts("Failed to commit localization files.. maybe none to commit?")
    end

    puts("Switching to back to branch: #{build_branch} to continue building project.")
    sh('git', 'checkout', build_branch)
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

    unless ENV['CRU_SKIP_LOCALIZATION_DOWNLOAD'].present?
      cru_download_localizations
    end

    automatic_code_signing(
        use_automatic_signing: false,
        profile_name: profile_name
    )

    cru_fetch_certs(type: type)

    unless ENV["CRU_SKIP_COCOAPODS"].present?
      cocoapods(
          podfile: './Podfile'
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
    if is_ci?
      create_keychain(
          name: ENV["MATCH_KEYCHAIN_NAME"],
          password: ENV["MATCH_PASSWORD"],
          default_keychain: true,
          unlock: true,
          timeout: 3600,
          add_to_search_list: true
      )

      match(type: options[:type],
            keychain_name: ENV["MATCH_KEYCHAIN_NAME"],
            keychain_password: ENV["MATCH_PASSWORD"])
    else
      match(type: options[:type])
    end
  end

  lane :cru_update_commit do |options|
    clean_build_artifacts

    if is_ci?
      travis_branch = ENV["TRAVIS_BRANCH"]

      sh('git', 'checkout', 'master')
      sh('git', 'pull', 'origin', 'master')
    end

    commit_version_bump(
        xcodeproj: ENV["CRU_XCODEPROJ"],
        message: options[:message]
    )

    push_to_git_remote
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

