# Flutter configuration

## Take note

- It contains 3 main steps: linting, testing and build automaticcaly
- Support for Gitlab(Gitlab Action), Bitbucket(Bitbucket pipeline), Gitlab(Gitlab Action)
- Focusing on Android first because it's easier to to start with.
- Some options for IOS vertion:
  [Codemagic](https://codemagic.io/)
  [Bitrise](https://www.bitrise.io/)
  [Fastlane](https://fastlane.tools/)
  A mac computer is always open to build (best way)

## Configuration with Gitlab Action:

- In the root repository, create `.github` directory and `workflow` sub directory
- Create main.yml file which contains configuration for ci/cd
- Writing commands to identify what will occur in process
- Here is my configuration:

```
on: pull_request
name: Build Debug and Release APK
jobs:
  build:
    name: Build APK
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - uses: actions/setup-java@v1
        with:
          java-version: "12.x"
      - uses: subosito/flutter-action@v1
        with:
          flutter-version: "1.7.8+hotfix.4"
      - run: flutter pub get
      - run: flutter test
      - run: flutter build apk --debug --split-per-abi
      - name: Build a debug APK
        uses: actions/upload-artifact@v1
        with:
          name: debug APK
          path: build/app/outputs/apk/debug/
      - run: flutter build apk --split-per-abi
      - name: Build a Release APK
        uses: actions/upload-artifact@v1
        with:
          name: release APK
          path: build/app/outputs/apk/release/
```

## Configuration with Bitbucket pipelines:

- In the root repository, create `bitbucket-pipelines.yml` file
- Docket file to configure more detail about flutter version,... author: Ha Quang Bao

```
FROM openjdk:8-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    curl \
    git \
    unzip \
    xz-utils \
    lib32stdc++6 \
    libglu1-mesa \
  && rm -rf /var/lib/apt/lists/*

# RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8

# Installing android base on what found at
# https://hub.docker.com/r/alvrme/alpine-android/~/dockerfile/

ENV SDK_TOOLS "3859397"
ENV BUILD_TOOLS "27.0.3"
ENV TARGET_SDK "27"
ENV ANDROID_HOME "/opt/sdk"

# Download and extract Android Tools
RUN curl -L http://dl.google.com/android/repository/sdk-tools-linux-${SDK_TOOLS}.zip -o /tmp/tools.zip --progress-bar && \
  mkdir -p ${ANDROID_HOME} && \
  unzip /tmp/tools.zip -d ${ANDROID_HOME} && \
  rm -v /tmp/tools.zip

# Install SDK Packages
RUN mkdir -p /root/.android/ && touch /root/.android/repositories.cfg && \
  yes | ${ANDROID_HOME}/tools/bin/sdkmanager "--licenses" && \
  ${ANDROID_HOME}/tools/bin/sdkmanager "--update" && \
  ${ANDROID_HOME}/tools/bin/sdkmanager "build-tools;${BUILD_TOOLS}" "platform-tools" "platforms;android-${TARGET_SDK}" "extras;android;m2repository" "extras;google;google$

# Install flutter
ENV FLUTTER_HOME "/opt/flutter"

ENV FLUTTER_VERSION "1.9.1+hotfix.6-stable"

RUN mkdir -p ${FLUTTER_HOME} && \
  curl -L https://storage.googleapis.com/flutter_infra/releases/stable/linux/flutter_linux_v${FLUTTER_VERSION}.tar.xz -o /tmp/flutter.tar.xz --progress-bar && \
  tar xf /tmp/flutter.tar.xz -C /tmp && \
  mv /tmp/flutter/ -T ${FLUTTER_HOME} && \
  rm -f /tmp/flutter.tar.xz

ENV PATH=$PATH:$FLUTTER_HOME/bin

```

- Set up command to define your work

```
image: duclinhclc/flutter-image:latest

pipelines:
  pull-requests:
    '**':
      - step:
          caches:
            - gradle
            - gradlewrapper
            - flutter
          deployment: test
          script:
            - echo "Show the configuration"
            - flutter doctor -v
            - echo "Running flutter analyze with linting rules"
            - flutter analyze
            - echo "Only on pull request"
            - echo "Building release APK..."
            - flutter -v build apk
            #            - SLACK_TOKEN=****
            - SLACK_CHANEL=jenkins
            - BRANCH_SLUG="   ${BITBUCKET_BRANCH//\//-}"
            - FILE_NAME="$BITBUCKET_REPO_SLUG$BRANCH_SLUG.apk"
            - cp build/app/outputs/apk/release/app-release.apk "build/app/outputs/apk/release/$FILE_NAME"
            - curl -F file=@"build/app/outputs/apk/release/$FILE_NAME" -F channels=${SLACK_CHANEL} -F token=${SLACK_TOKEN} https://slack.com/api/files.upload
          artifacts:
            - build/app/outputs/apk/release/app-release.apk
definitions:
  caches:
    gradlewrapper: ~/.gradle/wrapper
    flutter: /opt/flutter
```
