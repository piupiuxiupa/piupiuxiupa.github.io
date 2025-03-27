---
title: Gitlab download files
date: 2025-03-06 18:11:46
tags: linux
categories: Posts
---

# Gitlab 下载文件

背景：有需求从 gitlab 下载文件，但是又不想下载整个项目，而是只想下载某个目录下的所有文件。
     gitlab 上有些项目能达到几 G 到几十 G 不等，全部下载耗时。
     内网办公环境下无权限传入三方开源工具，所以只能重复造轮子，写了个 shell 版的临时使用。

主要用到三个接口：
- `api/v4/projects?search=${proj_name}` 查询项目信息接口
- `api/v4/projects/${proj_id}/repository/tree?ref=${branch}&path=${path}` 查询目录下文件信息接口
- `api/v4/projects/${proj_id}/repository/files/${path}/raw?ref=${branch}` 下载文件接口

大概思路：
1. 先通过 `api/v4/projects?search=${proj_name}` 查询项目信息接口，获取项目 id
2. 通过 `api/v4/projects/${proj_id}/repository/tree?ref=${branch}&path=${path}` 查询目录下文件信息接口，获取目录下所有文件路径
3. 遍历目录下所有文件路径，通过 `api/v4/projects/${proj_id}/repository/files/${path}/raw?ref=${branch}` 下载文件

```bash
#!/bin/bash

:<<'EOF'
API from gitlab version 17.4
EOF

function get_params {
    proj_name=`echo "$url" | awk -F'/-/' '{print $1}' | xargs basename`
    branch=`echo "$url" | awk -F'/-/' '{print $2}' | awk -F/ '{print $2}'`
    version_path=`echo "$url" | awk -F"/-/" '{print $2}' | awk -F "${branch}" '{print $2}'`
    gitlab_url=`echo "$url" | sed -E 's|^(https?://[^/]+).*|\1|'`
}

function encode_to_url {
    orl_name=${1##*/}
    file_name=${orl_name%.*}
    file_postfix=${orl_name##*.}

    test "${file_postfix}" = "${file_name}" && file_postfix=""

    file_name=`echo -n "${file_name}" | od -An -tx1 | tr ' ' % | tr -d "\n"`

    test -z "${file_postfix}" && file_name_plus=${file_name} || file_name_plus=${file_name}.${file_postfix}

}

function get_files {
    proj_id=$(curl -s -H "PRIVATE-TOKEN: ${TOKEN}" "${gitlab_url}/api/v4/projects?search=${proj_name}" \
        | jq -r --arg proj_name "${proj_name}" '.[] | select(.name==$proj_name) | .id')

    path_list=$(curl -s -H "PRIVATE-TOKEN: ${TOKEN}" \
        "${gitlab_url}/api/v4/projects/${proj_id}/repository/tree?ref=${branch}&path=${version_path#/*}" | jq -r ".[].path")

    version_path=`echo ${version_path#/*} | sed 's#/#%2F#g'`
    IFS=$'\n'
    for item in ${path_list}
    do
        encode_to_url "${item}"
        download_files
    done
}

function download_files {
    file_url="${gitlab_url}/api/v4/projects/${proj_id}/repository/files/${version_path}%2F${file_name_plus}/raw?ref=${branch}"
    curl -sSL -H "PRIVATE-TOKEN: ${TOKEN}" "${file_url}" -o "${orl_name}"
}

function options {
    while test $# -gt 0
    do
        case "$1" in
            -u|--url)
                url="$2"
                ;;
            -t|--token)
                TOKEN="$2"
                ;;
            *) Usage ;;
        esac
        shift 2
    done
}

Usage(){
    cat <<EOF
${0##*/} in order to download files from gitlab
    Usage:
        -u|--url       gitlab url.
        -t|--token     gitlab token.
EOF
}

main(){
    TOKEN=""
    options "$@"
    get_params
    get_files
}

main "$@"

exit 0
```