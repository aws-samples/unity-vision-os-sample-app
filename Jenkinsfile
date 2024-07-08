pipeline {
    agent none

    stages {
        stage('build Unity project on spot') {
            agent {
                docker {
                    label 'linux'
                    image "${ECR_REPOSITORY_URL}:latest"
                    args '-u 0 --entrypoint= '
                    registryCredentialsId "ecr:${AWS_REGION}:ecr-role"
                    registryUrl "$ECR_REGISTRY_URL"
                }
            }

            steps {
                sh '''#!/bin/bash
                set -xe
                printenv
                ls -la
                echo "===Installing stuff for unity"
                apt-get update
                apt-get install -y curl unzip zip jq

                # Create config file for acquiring Unity Build Server floating licenses:
                mkdir -p /usr/share/unity3d/config/
                echo '{
                "licensingServiceBaseUrl": "'"$UNITY_BUILD_SERVER_URL"'",
                "enableEntitlementLicensing": true,
                "enableFloatingApi": true,
                "clientConnectTimeoutSec": 5,
                "clientHandshakeTimeoutSec": 10}' > /usr/share/unity3d/config/services-config.json
                unity-editor \
                    -quit \
                    -batchmode \
                    -nographics \
                    -executeMethod ExportTool.ExportXcodeProject \
                    -projectPath "./"
                echo "===Zipping Xcode project"
                zip -q -r -0 visionOSBuild visionOSBuild
                '''
                // pick up archive xcode project
                dir('') {
                    stash includes: 'visionOSBuild.zip', name: 'xcode-project'
                }
            }
            post {
                always {
                    sh 'chmod -R 777 .'
                }
            }
        }
        stage('build and sign visionOS app on mac') {
            // we don't need the source code for this stage
            options {
                skipDefaultCheckout()
            }
            agent {
                label 'mac'
            }
            environment {
                PROJECT_FOLDER = 'visionOSBuild'
                CERT_PRIVATE = credentials('priv')
                CERT_SIGNATURE = credentials('development')
                BUILD_SECRET_JSON = credentials('visionos-build-secret')
            }
            steps {
                unstash 'xcode-project'
                sh '''
                set -xe
                printenv
                ls -l
                # Remove old project and unpack a new one
                sudo rm -rf ${PROJECT_FOLDER}
                unzip -q visionOSBuild.zip
                '''

                // create export options file
                writeFile file: "${env.PROJECT_FOLDER}/ExportOptions.plist", text: """
                <?xml version="1.0" encoding="utf-8"?>
                <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
                <plist version="1.0">
                <dict>
                    <key>signingStyle</key>
                    <string>manual</string>
                </dict>
                </plist>
                """

                sh '''
                # set -xe
                source ~/.zshrc
                export PATH=/usr/local/bin:/opt/homebrew/bin:\${PATH}

                cd ${PROJECT_FOLDER}
                TEAM_ID=$(echo $BUILD_SECRET_JSON | jq -r '.TEAM_ID')
                BUNDLE_ID=$(echo $BUILD_SECRET_JSON | jq -r '.BUNDLE_ID')
                # extra backslash for groovy
                sed -i "" "s/DEVELOPMENT_TEAM = \\"\\"/DEVELOPMENT_TEAM = $TEAM_ID/g" Unity-VisionOS.xcodeproj/project.pbxproj
                #############################################
                # setup certificates in a temporary keychain
                #############################################
                echo "===Setting up a temporary keychain"
                pwd
                # Unique keychain ID
                MY_KEYCHAIN="temp.keychain.`uuidgen`"
                MY_KEYCHAIN_PASSWORD="secret"
                security create-keychain -p "$MY_KEYCHAIN_PASSWORD" "$MY_KEYCHAIN"
                # Append the temporary keychain to the user search list
                # double backslash for groovy
                security list-keychains -d user -s "$MY_KEYCHAIN" $(security list-keychains -d user | sed s/\\"//g)
                # Output user keychain search list for debug
                security list-keychains -d user
                # Disable lock timeout (set to "no timeout")
                security set-keychain-settings "$MY_KEYCHAIN"
                # Unlock keychain
                security unlock-keychain -p "$MY_KEYCHAIN_PASSWORD" "$MY_KEYCHAIN"
                echo "===Importing certs"
                # Import certs to a keychain; bash process substitution doesn't work with security for some reason
                security -v import $CERT_SIGNATURE -k "$MY_KEYCHAIN" -T "/usr/bin/codesign"
                #rm /tmp/cert
                PASSPHRASE=""
                security -v import $CERT_PRIVATE -k "$MY_KEYCHAIN" -P "$PASSPHRASE" -t priv -T "/usr/bin/codesign"
                # Dump keychain for debug
                security dump-keychain "$MY_KEYCHAIN"
                # Set partition list (ACL) for a key
                security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k $MY_KEYCHAIN_PASSWORD $MY_KEYCHAIN
                # Get signing identity for xcodebuild command
                security find-identity -v -p codesigning $MY_KEYCHAIN
                # double backslash for groovy
                CODE_SIGN_IDENTITY=`security find-identity -v -p codesigning $MY_KEYCHAIN | awk '/ *1\\)/ {print $2}'`
                echo code signing identity is $CODE_SIGN_IDENTITY
                security default-keychain -s $MY_KEYCHAIN
                echo $MY_MY_KEYCHAIN > keychain.txt

                #############################################
                # Build
                #############################################
                echo ===Building
                pwd

                xcodebuild -scheme Unity-VisionOS -sdk xros -configuration AppStoreDistribution archive -archivePath "$PWD/build/Unity-VisionOS.xcarchive" CODE_SIGN_STYLE="Manual" CODE_SIGN_IDENTITY=$CODE_SIGN_IDENTITY OTHER_CODE_SIGN_FLAGS="--keychain=$MY_KEYCHAIN" -UseModernBuildSystem=0 CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO

                zip -r build/Unity-VisionOS.zip build/Unity-VisionOS.xcarchive
                '''
            }
            post {
                always {
                    sh '''
                    #############################################
                    # cleanup
                    #############################################
                    if [ -f "keychain.txt" ]; then
                        security delete-keychain $(cat keychain.txt)
                        rm keychain.txt
                    fi
                    '''
                    archiveArtifacts artifacts: 'visionOSBuild/build/Unity-VisionOS.zip', onlyIfSuccessful: true, caseSensitive: false
                }
            }
        }
    }
    post {
        success {
            echo 'Success ^_^'
        }
        failure {
            echo 'Failed :('
        }
    }
}
