# Linux Graphic Driver 설정 잡기

**✅사전 준비 사항**
- Ubuntu Server 22.0 LTS 버전 설치

<hr>

## ✏️ 그래픽 드라이브 다운로드
- OS 정보 확인
  ```bash
  lsb_release -a 
  ```
  ![](img/1.png)
  ```bash
  unmae -a 
  ```
  ![](img/2.png)
- 그래픽 정보 확인
  ```bash
  lspci | grep VGA
  ```
  ![](img/3.png)
- [NVIDIA Driver 다운로드](https://www.nvidia.com/download/index.aspx) <br>
  **위에서 찾은 정보 바탕으로 드라이버를 검색한다.**
  ![](img/4.png)
  ![](img/5.png)
  ![](img/6.png)
- run 파일 서버에 업로드
  ```bash
  scp NVIDIA-Linux-x86_64-550.107.02.run [계정명]@[host]:[옮길 위치]
  ```
  ![](img/7.png)