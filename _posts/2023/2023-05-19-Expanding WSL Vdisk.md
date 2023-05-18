---
category: 2023
tags: ["2023", "wsl", "vdisk", "diskpart", "ubuntu", "windows", "linux"]
---

# WSL Vdisk 공간을 확장하는 방법

스테이블 디퓨전 관련 작업을 하다 보니까 WSL의 디스크 공간이 꽉 차는 일이 발생했다. 사용하고 있는 SSD가 2TB라서 자동으로 공간이 확장되는 줄 알았는데, 아니었다. 256GB의 기본 공간이 제공되고 있었고 이게 가득 차니까 no disk space 에러가 났다.

WSL의 디스크 공간을 확장하기 위해서 [마이크로소프트 공식 문서](https://learn.microsoft.com/ko-kr/windows/wsl/disk-space)를 참고했다.

## 1. WSL의 디스크 공간 확인하기

```powershell
> wsl --shutdown
> wsl --list
Linux용 Windows 하위 시스템 배포:
Ubuntu-22.04(기본값)
docker-desktop
docker-desktop-data
```

```powershell
> wsl --system -d Ubuntu-22.04 df -h /mnt/wslg/distro
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc        251G  238G  124M 100% /mnt/wslg/distro
```

## 2. WSL의 디스크 위치 확인하기

```powershell
> (Get-ChildItem -Path HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss | Where-Object { $_.GetValue("DistributionName") -eq 'Ubuntu-22.04' }).GetValue("BasePath") + "\ext4.vhdx"
C:\Users\hoya\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu22.04LTS\LocalState\ext4.vhdx
```

여기서 나온 파일 경로를 메모장에 복사해 둔다.

## 3. WSL의 디스크 공간 확장하기

```powershell
diskpart
Select vdisk file="C:\Users\hoya\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu22.04LTS\LocalState\ext4.vhdx"
detail vdisk
expand vdisk maximum=1024000
exit
```

1024000MB 즉 1TB의 공간을 할당해준다. 주의할 점은 마이크로소프트 공식 문서에도 나온 것과 같이 가상 디스크 크기를 줄이는 프로세스가 훨씬 더 복잡하기 때문에 실제로 원하는 것보다 높은 값을 입력하지 않도록 주의한다.

이제 가상 디스크 파일은 공간이 확장되었고, 이걸 리눅스 파일시스템에 인식시켜줘야 한다. 리눅스 터미널에서 아래 명령을 입력한다.

```bash
sudo mount -t devtmpfs none /dev
mount | grep ext4
sudo resize2fs /dev/sdc 1024000M
```

mount 명령은 오류가 날 수 있다고 하는데 무시하라고 한다.

루트 디렉토리가 마운트된 장치 파일명(내 경우 /dev/sdc) 에 대해 `resizefs`명령을 실행한다.

이제 WSL의 디스크 공간이 확장되었다. 하지만 기본적으로 가상 디스크 파일은 sparse file이기 때문에 1TB로 확장하더라도 실제 디스크 공간을 그만큼 차지하지는 않는다. 다만 한 번 늘어났던 공간이 자동으로 다시 줄어들지는 않으므로 주의한다.

## 번외: WSL의 디스크 공간 줄이기

당연히 실제 파일이 차지하는 공간을 줄이는 것은 아니고 파일이 삭제된 뒤의 빈 공간을 줄이는 것이다.

[https://stephenreescarter.net/how-to-shrink-a-wsl2-virtual-disk/](https://stephenreescarter.net/how-to-shrink-a-wsl2-virtual-disk/)

```powershell
wsl --shutdown
diskpart
Select vdisk file="C:\Users\hoya\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu22.04LTS\LocalState\ext4.vhdx"
compact vdisk
```

몇몇 블로그에서는 이걸 하기 전에 `sdelete -z` 명령을 실행하라고 하는데, 실행 시간도 오래 걸릴 뿐더러 실행 직후 VDISK의 크기가 설정된 최대 용량까지 올라갈 것이다. VDISK상에서 파일이 삭제될 때 제대로 TRIM이 실행되었다면(확실하지는 않다. WSL이 TRIM을 지원하는지도 모르겠다.) 이걸 실행할 필요가 없다고 생각한다.
