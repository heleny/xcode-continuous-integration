## Xcode Continuous Integration 

**Notes for myself**

Here is Apple's guide on [Xcode Continuous Integration Guide].

How to setup Xcode continuous integration using Xcode Server

* Install and configure [OS X Server]
* Install the latest [Xcode]
* Create a Bot in Xcode
* Upload the ipa to [Amazon S3] or TestFlight using [s3cmd]
* Set up access_key and secret_key of the aws account

```markdown
		$s3cmd --configuration
```

```markdown
		s3cmd="python ${HOME_DIRECTORY}/dev/s3cmd-master/s3cmd --access_key=xxxxxxxxxx --secret_key=xxxxxxxxxx"
		$s3cmd put ${XCS_OUTPUT_DIR}/${XCS_BOT_NAME}.ipa s3://${FOLDER_NAME}/${APPLICATION_NAME}.ipa	
```

* For uploading *.ipa to TestFlight, see details here [Upload Xcode Build to TestFlight]
* If bot build fails somehow, make sure to check out [Xcode Bots Common Problems And Workarounds].

  My bots kept failing when I forgot to copy my keys to System instead of Login on Keychain.  <br/>
  Make sure to download the Apple Production IOS Push Services certificate to the Keychain Access.

* In order to enable Apple Push Notification for Bot builds, make sure you have setup 'Code Signing Entitlements'
* Create Entitlements.plist if not available, it would look like something similar to the following: 

```markdown
<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
	<plist version="1.0">
	<dict>
        	<key>com.apple.developer.team-identifier</key>
        	<string>${APPLE_DEVELOPER_ACCOUNT_TEAM_ID}</string> 
		<key>application-identifier</key>
       		<string>${APP_ID_PREFIX}.${XCODE_PROJECT_BUNDLE_IDENTIFIER}</string>  // project target -> General -> Identify -> Bundle Identifier 
        	<key>aps-environment</key>
        	<string>production</string> // or development
        	<key>keychain-access-groups</key>
        	<array>
                	<string>${APP_ID_PREFIX}.*</string>
        	</array>
	</dict>
	</plist>
```

* Click on project target -> Build Settings -> Code Signing -> add the path of Entitlements.plist to 'Code Signing Entitlements'.

  It took me a couple of days to figure this out.  Thanks to these posts on stackoverflow [here] and [there].
* Steps to verify entitlements is picked by the code signing process: 
	* unzip *.ipa file
	* cd Payload/*.app/
	* open your app binary file and look for aps-environment
	* if aps-environment is not in the binary file but it's found in the embedded.mobileprovision, double check Entitlements.plist
* If push notification still doesn't work, you can try to re-sign your ipa

```markdown       
		SIGNING_IDENTITY="xxxxxxxxxxxxxxxx.mobileprovision"
		PROVISIONING_PROFILE="${HOME}/Library/MobileDevice/Provisioning Profiles/xxxxx_adhoc.mobileprovision"
		/usr/bin/xcrun -sdk iphoneos PackageApplication -v "${RELEASE_BUILDDIR}/${APPLICATION_NAME}.app" -o "${BUILD_HISTORY_DIR}/${APPLICATION_NAME}.ipa" --sign "${SIGNING_IDENTITY}" --embed "${PROVISONING_PROFILE}"
```

* echo set using bot script by editing the bot, [access-xcode-server-bot-run-envariables] is extremely helpful.

```markdown
		set
```

* Once the build is done, click on Logs, all the env are listed there


Notes:
You don’t need the entitlements file for a development build using a development provisioning profile unless you want to use push notifications, iCloud storage, or keychain sharing.  
To enable these features you must specify an entitlements file during code signing. Entitlements file must contain the application-identifier and aps-environment keys.

[OS X Server]: https://www.apple.com/ca/support/osxserver/setupadministration/ 
[Xcode]: https://developer.apple.com/xcode/downloads/
[Amazon S3]: http://aws.amazon.com/s3/
[s3cmd]: https://github.com/s3tools/s3cmd
[Xcode Bots Common Problems And Workarounds]: http://ikennd.ac/blog/2013/10/xcode-bots-common-problems-and-workarounds/
[Upload Xcode Build to TestFlight]: http://www.developmentseed.org/blog/2011/sep/02/automating-development-uploads-testflight-xcode/
[here]: http://stackoverflow.com/questions/10987102/how-to-fix-no-valid-aps-environment-entitlement-string-found-for-application
[there]: http://stackoverflow.com/questions/21947261/ipa-created-via-xcode-bot-fails-to-run-for-apns-but-runs-if-built-manually-via-x
[access-xcode-server-bot-run-envariables]: http://stackoverflow.com/questions/25127146/access-build-folder-in-xcode-server-ci-bot-run-env-varaibles
[Xcode Continuous Integration Guide]: https://developer.apple.com/library/ios/documentation/IDEs/Conceptual/xcode_guide-continuous_integration/Xcode_Continuous_Integration_Guide.pdf

## FAQ on Xcode Server and Bot ##

**Q**: No matching provisioning profile found: Your build settings specify a provisioning profile with the UUID “XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX”, however, no such provisioning profile was found.
CodeSign error: code signing is required for product type 'Application' in SDK 'iOS 8.1'.<br/>
**A**: Copy provisioning profiles from ~/Library/MobileDevice/Provisioning\ Profiles/ to /Library/Developer/XcodeServer/ProvisioningProfiles/. 

**Q**: Hwo to remove all traces of ever running Xcode Server on your system?<br/>
**A**: ```sudo xcrun xcscontrol --reset``` 

**Q**: If you're hosting the Xcode server on a mac mini, what is the easiest and quickest way to remote access your Server?<br/>
**A**: ssh into your server, and then run the following
```markdown
		sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -activate -configure -access -on -clientopts -setvnclegacy -vnclegacy yes -clientopts -setvncpw -vncpw traderev -restart -agent -privs -all
		open /System/Library/CoreServices/Screen\ Sharing.app/
```
Once done, turn it off
```markdown
		sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -deactivate -configure -access -off
```

**Q**: How to fix xcode build server's internal-checkout-error?<br/>
**A**: A couple of things need to check: 
- submodule detached from HEAD
	- check Xcode -> Source Control
	- make sure all working branch are pointing to master branch
	
- Error Domain=XCSBuildServiceDomain Code=-1 "SCM dictionary scmType is nil"
	- make sure all working copies under Xcode -> Source Control are from the same workspace
	- make sure all submodules are in the same workspace of the TradeRev
	- otherwise, clean it up 
	- if it still doesn't work,  clean server cache:
		rm -rf ~/Library/Developer/Xcode/DerivedData/
		rm -rf /Library/Server/Xcode/Data/BotRuns
		rm -f /Library/Server/Xcode/Data/BotRuns/Cache

**Q**: Why Bot doesn't show up correctly on Xcode?<br/>
**A**: Try to delete all the repos and server via Xcode -> Preferences -> Accounts, and then add them back, do a clean on /Library/Server cache and bots, and flush out Xcode's DerivedData. 

**Q**: Where is the *.ipa created by Xcode Bot?<br/>
**A**: /Library/Server/Xcode/Data/BotRuns/BotRun-ec531f8a-8501-486a-84ad-98045f03f0a2.bundle/output/ibot.ipa
