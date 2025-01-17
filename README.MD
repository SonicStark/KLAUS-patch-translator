# KLAUS-patch-translator

Learn patch analysis techniques from KLAUS :)

## Dockerfile restore

```docker
FROM ubuntu:22.04

RUN /bin/sh -c apt update

RUN /bin/sh -c apt install -y vim python3 python3-pip unzip libz3-4 git flex bison libssl-dev bc qemu-system

RUN /bin/sh -c pip3 install requests

COPY klaus_fuzzer.zip llvm.zip gcc.zip analyzer.zip image.zip /

RUN /bin/sh -c unzip klaus_fuzzer.zip && \
    unzip llvm.zip && \
    unzip gcc.zip && \
    unzip analyzer.zip && \
    unzip image.zip

RUN /bin/sh -c apt-get update
RUN /bin/sh -c apt-get install wget

RUN /bin/sh -c wget https://storage.googleapis.com/syzkaller/stretch.img -O /image/stretch.img
RUN /bin/sh -c wget https://storage.googleapis.com/syzkaller/stretch.img.key -O /image/stretch.id_rsa
RUN /bin/sh -c chmod 0600 /image/stretch.id_rsa
```

## `build_env.py` workflow

Run with
```py
commitid = "730c5fd42c1e" # The commit id of the buggy patch
syzid = "53b6555b27af2cae74e2fbdac6cadc73f9cb18aa" # The bug report id of the bug that the patch fixed
```

we got
```bash
# proj_dir   = /kernels/730c5fd42c1e
# commitid   = 730c5fd42c1e3652a065448fd235cb9fafb2bd10
# commit_url = https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux
# syzpoc     = https://syzkaller.appspot.com/text?tag=ReproSyz&x=10117666600000
# config     = https://syzkaller.appspot.com/text?tag=KernelConfig&x=5cbaa3be0b36022f
# patchlink  = https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/patch/?id=730c5fd42c1e3652a065448fd235cb9fafb2bd10

mkdir -p ${proj_dir}/linux_bug

# write_env_files

curl ${config}    > ${proj_dir}/config
curl ${patchlink} > ${proj_dir}/buggy.patch

# prepare_kernels

cd ${proj_dir}/linux_bug

git init
git remote add origin ${commit_url}
git fetch origin ${commitid}
git reset --hard FETCH_HEAD

cd ${proj_dir}

cp -r linux_bug linux_buggy_patched
cp -r linux_bug linux_buggy_patched_for_fuzz

cd ${proj_dir}/linux_bug

patch -R -p1 < ../buggy.patch

cd ${proj_dir}

cp config ./linux_bug/.config
cp config ./linux_buggy_patched/.config
cp config ./linux_buggy_patched_for_fuzz/.config

cd ${proj_dir}/linux_bug

./scripts/config -d CONFIG_KASAN -d CONFIG_KCOV -d CONFIG_UBSAN -d CONFIG_KTSAN

cd ${proj_dir}/linux_buggy_patched

./scripts/config -d CONFIG_KASAN -d CONFIG_KCOV -d CONFIG_UBSAN -d CONFIG_KTSAN

# compile_bitcodes

cd ${proj_dir}/linux_bug
make clean
yes "" | make CC="/llvm-project-10.0.1/build/bin/clang" -j`nproc`

cd ${proj_dir}/linux_buggy_patched
make clean
yes "" | make CC="/llvm-project-10.0.1/build/bin/clang" -j`nproc`

# analyze_patch

timeout 6h python3 /patch_analyzer/analyze_patch.py ${proj_dir}/buggy.patch

# compile_fuzzing_kernel

cd ${proj_dir}/linux_buggy_patched_for_fuzz

make clean
patch ${proj_dir}/linux_buggy_patched_for_fuzz/kernel/kcov.c ./kernel.patch
yes "" | COND_FILE=${proj_dir}/cond.txt PROP_FILE=${proj_dir}/prop.txt make CC="/gcc-bin/bin/gcc" -j`nproc`

# build_fuzzing_env

mkdir -p ./${commitid}
mkdir -p /fuzz_workdir/workdir_${commitid}

curl ${syzpoc} > ./${commitid}/poc
cp ./config ./${commitid}/config

sed -i "s/\[id\]/${commitid}/g"                           ./${commitid}/config
sed -i "s/\[port\]/$((10000 + RANDOM * 50000 / 32767))/g" ./${commitid}/config

mkdir -p ./${commitid}/db
cp ./${commitid}/poc ./${commitid}/db
/klaus_fuzzer/bin/syz-db pack ./${commitid}/db ./${commitid}/corpus.db

# fuzz_start

cd ./${commitid}
timeout 3d /klaus_fuzzer/bin/syz-manager -config $(pwd)/config -auxiliary $(pwd)/poc-auxiliary 2>&1
```

However
 - https://github.com/wupco/KLAUS/blob/main/Docker-env/fuzz_cfgs_dir/build_env.py
 - https://github.com/wupco/KLAUS/blob/main/Syzpatch/setup_env/build_env.py

are different in some lines. And the role of 
 - https://github.com/wupco/KLAUS/blob/main/Docker-env/fuzz_cfgs_dir/clang.patch
 - https://github.com/wupco/KLAUS/blob/main/Docker-env/fuzz_cfgs_dir/classmap.patch

is still unclear...
