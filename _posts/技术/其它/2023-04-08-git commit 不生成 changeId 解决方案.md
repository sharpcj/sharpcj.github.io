---
layout: post
title: "git commit 不生成 changeId 解决方案"
date:   2023-04-08 15:36:07 +0800
categories: ["技术", "其它"]
tag: ["git", "账号"]
---

1. 检查仓储 `.git/hook` 下面是否有 `commit-msg` 文件，如果没有可以到下面的地址下载，或者把其他同事的 `commit-msg` 文件拷贝到你的 `.git/hook` 重新 commit 即可。

[https://review.cyanogenmod.org/tools/hooks/commit-msg](https://review.cyanogenmod.org/tools/hooks/commit-msg)
[https://gerrit-review.googlesource.com/tools/hooks/commit-msg](https://gerrit-review.googlesource.com/tools/hooks/commit-msg)

如果有自己的 gerrit-review 服务器，可以直接在网址后面加上 `/tools/hooks/commit-msg` 即可下载。

添加后，每次执行git commit 都会自动在log里面生成 Change-Id，用于gerrit code review。

注意：下载 `commit-msg` 需要设置执行权限：#**chmod a+x .git/hook/commit-msg**

2. 如果是repo sync 下来的代码，随便找一个仓储，按上面的方法，检查是否存在 commit-msg 软链接（repo sync 是在每个仓储 .git/hooks 下面创建的软链接），如果不存在，修改工程目录下面 `.repo/manifest.xml`，注意这个xml文件也是软链接。

```
<remote  name="aosp" review="review.source.android.com" fetch=".." />
<default revision="master" remote="aosp" sync-j="4" />
```

注意必须添加上面的 review="review.source.android.com" 这句。至于为什么，可以查看.repo/repo 下面的 python脚本。

附： commit-msg

```
#!/bin/sh
# From Gerrit Code Review 3.6.0-rc4-360-g5a87dddd18
#
# Part of Gerrit Code Review (https://www.gerritcodereview.com/)
#
# Copyright (C) 2009 The Android Open Source Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set -u

# avoid [[ which is not POSIX sh.
if test "$#" != 1 ; then
  echo "$0 requires an argument."
  exit 1
fi

if test ! -f "$1" ; then
  echo "file does not exist: $1"
  exit 1
fi

# Do not create a change id if requested
if test "false" = "$(git config --bool --get gerrit.createChangeId)" ; then
  exit 0
fi

if git rev-parse --verify HEAD >/dev/null 2>&1; then
  refhash="$(git rev-parse HEAD)"
else
  refhash="$(git hash-object -t tree /dev/null)"
fi

random=$({ git var GIT_COMMITTER_IDENT ; echo "$refhash" ; cat "$1"; } | git hash-object --stdin)
dest="$1.tmp.${random}"

trap 'rm -f "${dest}"' EXIT

if ! git stripspace --strip-comments < "$1" > "${dest}" ; then
   echo "cannot strip comments from $1"
   exit 1
fi

if test ! -s "${dest}" ; then
  echo "file is empty: $1"
  exit 1
fi

reviewurl="$(git config --get gerrit.reviewUrl)"
if test -n "${reviewurl}" ; then
  if ! git interpret-trailers --parse < "$1" | grep -q '^Link:.*/id/I[0-9a-f]\{40\}$' ; then
    if ! git interpret-trailers \
          --trailer "Link: ${reviewurl%/}/id/I${random}" < "$1" > "${dest}" ; then
      echo "cannot insert link footer in $1"
      exit 1
    fi
  fi
else
  # Avoid the --in-place option which only appeared in Git 2.8
  # Avoid the --if-exists option which only appeared in Git 2.15
  if ! git -c trailer.ifexists=doNothing interpret-trailers \
        --trailer "Change-Id: I${random}" < "$1" > "${dest}" ; then
    echo "cannot insert change-id line in $1"
    exit 1
  fi
fi

if ! mv "${dest}" "$1" ; then
  echo "cannot mv ${dest} to $1"
  exit 1
fi
```