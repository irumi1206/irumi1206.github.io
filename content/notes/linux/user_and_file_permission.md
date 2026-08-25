---
title: "User and File Permission"
hideSummary: true
---

### User/Group and File Permissions

Linux에서는 여러 user가 하나의 system을 사용할 수 있으며, kernel은 각각의 user를 UID(User ID)로 구분한다. `root`는 UID가 0인 특별한 administrative user이며, 일반 user보다 훨씬 넓은 권한을 가진다. 일반 user는 필요한 경우 `sudo`를 사용하여 특정 command를 다른 user의 권한으로 실행할 수 있으며, 기본적으로는 root 권한으로 실행된다. User는 하나 이상의 group에 속할 수 있고 group은 GID(Group ID)로 구분된다. Ubuntu에서는 일반적으로 user를 생성할 때 같은 이름의 primary group이 함께 만들어지는 경우가 많으며, 이후 필요에 따라 여러 supplementary group에 추가로 속할 수 있다.

Linux의 file과 directory에는 각각 owner user 하나와 group owner 하나가 지정되어 있으며, permission은 `owner / group / others` 세 범주에 대해 정의된다. 여기서 owner는 해당 file이나 directory의 소유 user, group은 해당 object에 지정된 하나의 group owner, others는 그 외의 user를 의미한다. 접근하는 user가 owner라면 owner permission이 적용되고, owner는 아니지만 그 user가 속한 primary 또는 supplementary group 중 하나가 file의 group owner와 일치하면 group permission이 적용된다. 둘 다 해당하지 않으면 others permission이 적용된다.

각 범주에는 `r`, `w`, `x` permission이 존재하며 `r`은 read, `w`는 write, `x`는 execute를 의미하고 숫자로는 각각 4, 2, 1로 표현할 수 있다. 따라서 `644`는 `rw-r--r--`, `755`는 `rwxr-xr-x`를 의미하며 `chmod`를 통해 permission을 변경할 수 있다. File과 directory에서는 `rwx`의 의미가 조금 다르다. 일반 file에서 `r`은 내용을 읽는 권한, `w`는 내용을 수정하는 권한, `x`는 executable로 실행할 수 있는 권한을 의미한다. Directory에서 `r`은 directory 안의 entry 이름을 조회하는 권한, `w`는 내부에 file이나 directory를 생성·삭제·rename할 수 있는 권한, `x`는 해당 directory를 통과하여 내부 path에 접근할 수 있는 권한을 의미한다.

다음 command들을 통해 현재 user와 group, file 및 directory permission을 직접 확인해보면

```bash
whoami
# sungha

id
# uid=1000(sungha) gid=1000(sungha) groups=1000(sungha),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)

groups
# sungha adm cdrom sudo dip plugdev users lpadmin

touch permission_test.txt
mkdir permission_test_dir

ls -l permission_test.txt
# -rw-rw-r-- 1 sungha sungha ... permission_test.txt

ls -ld permission_test_dir
# drwxrwxr-x 2 sungha sungha ... permission_test_dir

stat permission_test.txt
# 접근: (0664/-rw-rw-r--)  UID: (1000/sungha)  GID: (1000/sungha)

chmod 644 permission_test.txt
ls -l permission_test.txt
# -rw-r--r--

chmod 755 permission_test.txt
ls -l permission_test.txt
# -rwxr-xr-x

chmod u-x permission_test.txt
ls -l permission_test.txt
# -rw-r-xr-x

touch permission_test_dir/file.txt

chmod 700 permission_test_dir
ls -ld permission_test_dir
# drwx------

chmod 600 permission_test_dir
ls permission_test_dir
# file.txt 이름은 보이지만 내부 entry 접근 과정에서 Permission denied 발생

cd permission_test_dir
# Permission denied

chmod 700 permission_test_dir
cd permission_test_dir
# 정상적으로 directory 내부로 이동 가능
```

`id` 출력에서 `gid=1000(sungha)`는 현재 user의 primary group을 의미하며, `adm`, `sudo`, `plugdev` 등은 추가로 속해 있는 supplementary group이다. 하나의 user는 하나의 primary group과 여러 supplementary group에 속할 수 있다. 예를 들어 `ls -l`에서 `sungha sungha`가 보인다면 첫 번째 `sungha`는 owner user이고 두 번째 `sungha`는 group owner이다. 만약 어떤 file의 group owner가 `sudo`라면, owner가 아닌 user라도 `sudo` group에 속해 있을 경우 group permission이 적용된다. 반대로 두 user가 다른 group을 공통으로 가지고 있더라도 그 group이 해당 file의 group owner가 아니라면 그 file의 group permission과는 관계가 없다.

`ls -l`의 출력에서 첫 번째 문자는 object의 종류를 나타내며 `-`는 regular file, `d`는 directory를 의미하고, 그 뒤의 9개 문자는 세 글자씩 owner, group, others의 permission을 나타낸다.
