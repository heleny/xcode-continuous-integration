xcode-continuous-integration 

**Notes for myself**

How to setup Xcode continuous integration using Xcode Server

- Install and configure [OS X Server]
- Install the latest [Xcode]
- Create a Bot in Xcode
- Upload the ipa to [Amazon S3] or TestFlight using [s3cmd]
	- set up access_key and secret_key of the aws account
```
		$s3cmd --configuration
```
```
		s3cmd="python ${HOME_DIRECTORY}/dev/s3cmd-master/s3cmd --access_key=xxxxxxxxxx --secret_key=xxxxxxxxxx"
		$s3cmd put ${XCS_OUTPUT_DIR}/${XCS_BOT_NAME}.ipa s3://${FOLDER_NAME}/${APPLICATION_NAME}.ipa	
```
- For uploading to TestFlight, see details here [Upload Xcode Build to TestFlight]
- If bot build fails somehow, make sure to check out [Xcode Bots Common Problems And Workarounds]
  My bots kept failing when I forgot to copy my keys to System instead of Login on Keychain
- In order to enable Apple Push Notification for Bot build, make sure you have setup 'Code Signing Entitlements'
	- create Entitlements.plist, which will look like something similar to the following: 
<pre><code>
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
</pre></code>
        - Click on project target -> Build Settings -> Code Signing -> add the path of Entitlements.plist to 'Code Signing Entitlements'
        It took me a couple days to figure this out.  Thanks to these posts on stackoverflow [here] and [there].
        - Steps to verify entitlements is picked by the code signing process: 
			* unzip *.ipa file
			* cd Payload/*.app/
			* open your app binary file and look for aps-environment
			* if aps-environment is not in the binary file but it's found in the embedded.mobileprovision, double check Entitlements.plist
        - If push notification still doesn't work, you can try to re-sign your ipa
        
```
	SIGNING_IDENTITY="xxxxxxxxxxxxxxxx.mobileprovision"
	PROVISIONING_PROFILE="${HOME}/Library/MobileDevice/Provisioning Profiles/xxxxx_adhoc.mobileprovision"
	/usr/bin/xcrun -sdk iphoneos PackageApplication -v "${PRODUCT_NAME}.app" -o "/tmp/${PRODUCT_NAME}.ipa" --sign "${SIGNING_IDENTITY}" --embed "${PROVISIONING_PROFILE}"
```
	- echo set using bot script by editing the bot, [access-xcode-server-bot-run-envariables] is extremely helpful.
```
	set
```
	- Once the build is done, click on Logs, all the env are listed there


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
