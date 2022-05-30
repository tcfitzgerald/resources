# Notes for creating package for upload to Mac App store

### Create .pkg file for uploading via Transport app
```
productbuild --component PATH_TO_.APP_FILE /Applications --sign "3rd Party Mac Developer Installer: NAME ON CERT (TEAM_ID)" PACKAGE_NAME.pkg
```

# Notes for notarizing app

### Codesign the app

```
codesign --deep --force --verify --verbose --sign "Developer ID Application: NAME ON CERT (TEAM_ID)" --options runtime "PATH_TO_.APP_FILE"
```

### Zip the .app file up

```
/usr/bin/ditto -c -k --keepParent APP_FILE APP_MAJOR.MINOR.PATCH.zip
```

### Submit zip file

```
xcrun notarytool submit APP_MAJOR.MINOR.PATCH.zip --keychain-profile "MAC_NOTARIZE"
```

```MAC_NOTARIZE``` here is a keychain item created to store an App Password.  You can create this from the command line:

```
xcrun notarytool store-credentials "MAC_NOTARIZE"
               --apple-id "APPLE_ID"
               --team-id <TeamID>
               --password <secret_2FA_password>
```

### Check on status

```
xcrun notarytool history --keychain-profile "MAC_NOTARIZE"
```

### Staple notarization to .app bundle

```
xcrun stapler staple -v "PATH_TO.APP"
```
