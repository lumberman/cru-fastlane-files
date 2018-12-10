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
        submit_for_review: true,
        submission_information: cru_submission_information
    )

    cru_bump_version_number(version_number: version_number)

    cru_notify_users(message: "#{target} iOS Release Build #{version_number} (#{build_number}) submitted to App Store.")
    cru_notify_users(message: "Build has been submitted for review and will be #{automatic_release ? 'automatically' : 'manually'} released.")

  end

  desc "Push a new (beta) release build to TestFlight"
  lane :beta do
    target = ENV["CRU_TARGET"]

    build_number = cru_set_build_number
    version_number =  get_version_number(
        target: target
    )
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

    unless ENV["CRU_CALLDIRECTORY_TARGET"].nil?
      call_directory_profile  = type == "adhoc" ? ENV["CRU_CALLDIRECTORY_ADHOC_PROFILE_NAME"] : ENV["CRU_CALLDIRECTORY_APPSTORE_PROFILE_NAME"]
      automatic_code_signing(
          use_automatic_signing: false,
          targets: ENV["CRU_CALLDIRECTORY_TARGET"],
          profile_name: call_directory_profile
      )
    end

    cru_fetch_certs(type: type)

    if ENV["CRU_SKIP_COCOAPODS"].nil?
      cocoapods(
          podfile: './Podfile',
          try_repo_update_on_error: true
      )
    end

    gym(
	suppress_xcode_output: true,
        silent: true,
	scheme: ENV["CRU_SCHEME"],
        export_method: export_method,
        export_options: {
            provisioningProfiles: {
                ENV["CRU_APP_IDENTIFIER"] => profile_name
            }
        }
    )
  end

  lane :cru_build_adhoc do |options|
    unless ENV["CRU_ADHOC_PROFILE_NAME"].nil?
      target = ENV["CRU_TARGET"]
      version_number =  get_version_number(
        target: target
      )

      github_ipa_release_path = cru_build_app(profile_name: ENV["CRU_ADHOC_PROFILE_NAME"], type: "adhoc", export_method: "ad-hoc")
      cru_push_release_to_github(
        version_number: version_number,
        project_name: target,
        ipa_path: github_ipa_release_path
      )

      push_to_git_remote
    end
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

    unless ENV["CRU_CALLDIRECTORY_APP_IDENTIFIER"].nil?
      match(type: options[:type],
          username: ENV['CRU_FASTLANE_USERNAME'],
          app_identifier: ENV['CRU_CALLDIRECTORY_APP_IDENTIFIER'],
          keychain_name: ENV["MATCH_KEYCHAIN_NAME"],
          keychain_password: ENV["MATCH_PASSWORD"])
    end 
  end

  lane :cru_update_commit do |options|
    project_file_path = "#{ENV['CRU_XCODEPROJ']}/project.pbxproj"
    info_file_path = "#{ENV['CRU_SCHEME']}/Info.plist"
    git_commit(path:[project_file_path, info_file_path],
               message: options[:message])
  end

  lane :cru_bump_version_number do |params|
    if "v#{params[:version_number]}".eql? ENV['TRAVIS_TAG']

      sh('git remote set-branches --add origin master')
      sh('git fetch origin master:master')
      sh('git checkout master')
      version_number = increment_version_number(bump_type: 'patch')
      cru_update_commit(message: "[skip ci] Bumping version number to #{version_number} for next build")
      push_to_git_remote
    end
  end

  lane :cru_notify_users do |options|
    hipchat(
        message: options[:message],
        channel: ENV["HIPCHAT_CHANNEL"],
        version: "2",
        custom_color: "green"
    )
  end

  def cru_submission_information
    {
        export_compliance_available_on_french_store: false,
        export_compliance_contains_proprietary_cryptography: false,
        export_compliance_contains_third_party_cryptography: false,
        export_compliance_is_exempt: false,
        export_compliance_uses_encryption: false,
        export_compliance_app_type: nil,
        export_compliance_encryption_updated: false,
        export_compliance_compliance_required: false,
        export_compliance_platform: "ios",
        content_rights_contains_third_party_content: false,
        content_rights_has_rights: false,
        add_id_info_serves_ads: ENV['CRU_IDFA_SERVES_ADS'] || false,
        add_id_info_tracks_action: ENV['CRU_IDFA_TRACKS_ACTION'] || false,
        add_id_info_tracks_install: ENV['CRU_IDFA_TRACKS_INSTALL'] || false,
        add_id_info_uses_idfa: ENV['CRU_IDFA_IS_ENABLED'] || false
    }
  end

  lane :cru_push_release_to_github do |params|
    version = params[:version_number]
    project_name = params[:project_name]
    build = ENV["TRAVIS_BUILD_NUMBER"]
    ipa_path = params[:ipa_path]

    set_github_release(
        repository_name: ENV["TRAVIS_REPO_SLUG"],
        api_token: ENV["CI_USER_TOKEN"],
        name: "#{project_name} beta release  ##{version}-#{build}",
        tag_name: "#{version}-#{build}",
        description: "",
        commitish: ENV["TRAVIS_BRANCH"],
        upload_assets: [ipa_path],
        is_prerelease: true
    )
  end
end
