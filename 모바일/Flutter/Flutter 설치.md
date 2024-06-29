# Flutter 설치 및 환경설정(Mac)

<hr>

### Flutter SDK 설치

- 공식 홈페이지 접속 [링크](https://flutter.dev)
- 메인 우측 상단에 Get started 버튼 클릭
- 자신 원하는 플랫폼 선택합니다.(글쓴이는 macOS > iOS)
- Apple silicon Mac(M1 ~)은 Rosetta 2 를 설치해야 한다. 
![](./2/1.png)
- [Flutter SDK 설치에서 자신이 맞는 환경으로 선택한다.](https://docs.flutter.dev/get-started/install/macos/mobile-ios#install-the-flutter-sdk)
- zip 압축파일을 압축해제 합니다.
- 환경변수 파일(.zshrc)애 경로를 추가하고 저장한뒤 적용합니다.
~~~ shell
export PATH=$HOME/development/flutter/bin:$PATH

source [환경변수 파일]
~~~
- 저장한 뒤에 flutter 입력 하면 설치가 되고 flutter --version 입력하면 설치된 버전 정보가 보여집니다.
![](./2/2.png)

## Flutter 환경 체크

#### 플러터에서 개발 환경을 체크하는 방법이 있습니다.
~~~ shell
flutter doctor 
~~~
![](./2/3.png)
#### 결과 화면
- Android toolchain 이 안맞다.
- Xcode가 설치가 안되어있음
- Android Studio 가 설치되어 있지 않음

