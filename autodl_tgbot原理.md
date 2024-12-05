# 登录
往`https://www.autodl.com/api/v1/new_login`发送POST请求，请求体
```json
{
  "phone": "17610841208", 
  "password": "xxx",
  "v_code": "",
  "phone_area": "+86",
  "picture_id": null
}
```
其中phone为登录手机号，password为密码的**sha1**加密
POST报文部分（fetch格式）：
```
fetch("https://www.autodl.com/api/v1/new_login", {
  "headers": {
    "accept": "*/*",
    "accept-language": "zh-CN,zh;q=0.9",
    "appversion": "v5.56.0",
    "authorization": "null",
    "cache-control": "no-cache",
    "content-type": "application/json;charset=UTF-8",
    "pragma": "no-cache",
    "priority": "u=1, i",
    "sec-ch-ua": "\"Chromium\";v=\"130\", \"Google Chrome\";v=\"130\", \"Not?A_Brand\";v=\"99\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Windows\"",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "same-origin"
  },
  "referrer": "https://www.autodl.com/login",
  "referrerPolicy": "strict-origin-when-cross-origin"
```
接口会返回如下消息：
```json
{
    "code": "Success",
    "data": {
        "ticket": "xxx",
        "user": {
            "id": 369499,
            "created_at": "2024-04-04T17:26:19+08:00",
            "updated_at": "2024-04-04T17:26:19+08:00",
            "uuid": "17600253-79e1-4f29-b0f4-4a954f19c630",
            "username": "炼丹师1208",
            "nickname": "",
            "status": "normal",
            "logged_in_at": "2024-11-15T16:58:49.623+08:00",
            "email": "",
            "phone_area": "+86",
            "phone": "1***8",
            "is_admin": false,
            "is_super_admin": false,
            "backstage_role": "",
            "email_confirmed": false,
            "idle_job_authority": false,
            "mount_net_disk_authority": true,
            "bare_metal_authority": false,
            "invitation_code": "",
            "open_id": "xxx",
            "open_id_bind_at": "2024-04-04T17:28:31.715+08:00",
            "union_id": "",
            "applet_open_id": "",
            "avatar_id": 0,
            "register_ip": "1***5",
            "max_instance_num": 30,
            "max_region_instance_num": {},
            "sub_user_num": 10,
            "private_image_num": 20,
            "start_no_gpu_num": 0,
            "max_file_transfer_per_day": 0,
            "visible_adfs": false,
            "mount_adfs_disk_authority": false,
            "adfs_quota_size": "",
            "adfs_quota_inode": 0,
            "real_name_auth": false,
            "growth_value": 0,
            "member_level": "",
            "level_acq_mode": "",
            "member_dateline": null,
            "password_set": true,
            "update_password_key": "",
            "setting": {
                "max_clone_per_day": 0
            },
            "limit_recharge": false
        }
    },
    "msg": ""
}
```
需要进一步使用data的ticket字段。
# 获取token
往`https://www.autodl.com/api/v1/passport`发送POST请求，请求体：
```json
{"ticket":"xxx"}
```
POST报文部分（fetch格式）：
```
fetch("https://www.autodl.com/api/v1/passport", {
  "headers": {
    "accept": "*/*",
    "accept-language": "zh-CN,zh;q=0.9",
    "appversion": "v5.56.0",
    "authorization": "null",
    "cache-control": "no-cache",
    "content-type": "application/json;charset=UTF-8",
    "pragma": "no-cache",
    "priority": "u=1, i",
    "sec-ch-ua": "\"Chromium\";v=\"130\", \"Google Chrome\";v=\"130\", \"Not?A_Brand\";v=\"99\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Windows\"",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "same-origin"
  },
  "referrer": "https://www.autodl.com/login",
  "referrerPolicy": "strict-origin-when-cross-origin",
  "method": "POST",
  "mode": "cors",
  "credentials": "include"
});
```
接口返回如下信息：
```json
{
    "code": "Success",
    "data": {
        "token": "yyy"
    },
    "msg": ""
}
```
data的token字段需要用来请求之后的接口
# 获取当前机器
在访问控制台的实例页面后，页面会请求接口`https://www.autodl.com/api/v1/instance`，默认请求体
```json
{
  "date_from": "",
  "date_to": "",
  "page_index": 1,
  "page_size": 10,
  "status": [],
  "charge_type": []
}
```
POST报文部分（fetch格式），重点应该是authorization字段，使用的是之前获取到的token字段
```
fetch("https://www.autodl.com/api/v1/instance", {
  "headers": {
    "accept": "*/*",
    "accept-language": "zh-CN,zh;q=0.9",
    "appversion": "v5.57.0",
    "authorization": "xx",
    "cache-control": "no-cache",
    "content-type": "application/json;charset=UTF-8",
    "pragma": "no-cache",
    "priority": "u=1, i",
    "sec-ch-ua": "\"Chromium\";v=\"130\", \"Google Chrome\";v=\"130\", \"Not?A_Brand\";v=\"99\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Windows\"",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "same-origin"
  },
  "referrer": "https://www.autodl.com/console/instance/list",
  "referrerPolicy": "strict-origin-when-cross-origin",
  "body": "{\"date_from\":\"\",\"date_to\":\"\",\"page_index\":1,\"page_size\":10,\"status\":[],\"charge_type\":[]}",
  "method": "POST",
  "mode": "cors",
  "credentials": "include"
});
```
返回的结果
```json
{
    "code": "Success",
    "data": {
        "list": [
            {
                "created_at": "2024-10-04T17:54:17+08:00",
                "uuid": "2dee4c9145-b0f2fdc8",
                "machine_id": "2dee4c9145",
                "machine_alias": "641机",
                "region_sign": "beijing-E",
                "region_name": "北京A区",
                "rent_deadline": "2024-12-27T12:00:00+08:00",
                "has_auto_panel": true,
                "disk_expand_available": 321048805376,
                "driver_version": "525.105.17",
                "highest_cuda_version": "12.0",
                "status": "shutdown",
                "sub_status": "",
                "status_at": "2024-11-12T16:54:15+08:00",
                "oom_killed": false,
                "ssh_command": "xxx",
                "proxy_host": "xx",
                "proxy_host_ip": "",
                "snapshot_gpu_alias_name": "RTX 3090",
                "root_password": "SRfVpAGh+7xr",
                "jupyter_token": "xxx",
                "jupyter_port": 0,
                "tensorboard_port": 0,
                "jupyter_domain": "xx",
                "tensorboard_domain": "",
                "region_customer_port_visible": "enterprise",
                "ssh_port": 39451,
                "image": "xx",
                "private_image_uuid": "",
                "reproduction_uuid": "",
                "reproduction_id": 0,
                "cpu_limit": 15,
                "mem_limit_in_byte": 85899345920,
                "start_mode": "gpu",
                "shm_size": 42949672960,
                "charge_type": "payg",
                "req_gpu_amount": 1,
                "payg_price": 1577,
                "expired_at": {
                    "Time": "0001-01-01T00:00:00Z",
                    "Valid": false
                },
                "started_at": {
                    "Time": "2024-11-12T16:29:17+08:00",
                    "Valid": true
                },
                "stopped_at": {
                    "Time": "2024-11-12T16:54:09+08:00",
                    "Valid": true
                },
                "root_fs_used_rate": 0.3961,
                "disk_health_status": "normal",
                "cpu_usage_percent": 0,
                "mem_usage_percent": 0,
                "mem_usage": 0,
                "mem_limit": 0,
                "pull_image_progress": 0,
                "usage_info": {
                    "container_id": "autodl-container-2dee4c9145-b0f2fdc8",
                    "valid_at": "2024-11-12T16:29:29.539414816+08:00",
                    "cpu_usage_percent": 0,
                    "mem_usage_percent": 0,
                    "mem_usage": 0,
                    "mem_limit": 0,
                    "root_fs_used_size": 12759669484,
                    "root_fs_total_size": 27932976913,
                    "data_disk_total_size": 53687091200,
                    "data_disk_used_size": 118784,
                    "storage_fs_usage": "",
                    "pull_image_progress": 0,
                    "download_image_progress": 0,
                    "download_oss_file_progress": 0,
                    "is_new": false
                },
                "expand_data_disk_size": 0,
                "root_fs_used_size": 12759669484,
                "root_fs_total_size": 27932976913,
                "storage_fs_usage": "",
                "phone": "1***8",
                "name": "",
                "description": "",
                "timed_shutdown_at": {
                    "Time": "0001-01-01T00:00:00Z",
                    "Valid": false
                },
                "gpu_all_num": 8,
                "gpu_idle_num": 1,
                "machine_followed": 0,
                "auto_trans_to_payg": 0,
                "additional_info": {},
                "machine_describe": ""
            }
        ],
        "page_index": 1,
        "page_size": 10,
        "offset": 0,
        "max_page": 1,
        "result_total": 1,
        "page": 1
    },
    "msg": ""
}

```
List中每个代表一个实例的信息。
## GPU空闲
返回信息中，`gpu_all_num`和`gpu_idle_num`字段代表了当前实例的GPU总量和空闲量。而`region_name`和`machine_alias`可以指代特定机器
## 释放时间
返回信息中，`stopped_at`字段的`Time`表示上次停机的时间（格式为`2024-11-12T16:54:09+08:00`），停机后15天释放。也就是释放时间为：上次停机时间+15天-现在时间
## token认证
当没有携带authorization字段或者authorization字段不合法时，可能会返回如下信息
```json
{
    "code": "AuthorizeFailed",
    "data": null,
    "msg": "登陆超时，请重新登录; 登录超时，请重新登录"
}
```
# 开关机
## 开机
点击页面开机后，会先请求`detail`接口，确认后会请求`https://www.autodl.com/api/v1/instance/power_on`接口，请求体：
```json
{"instance_uuid":"11-11"}
```
这个`instance_uuid`与`instance`返回接口中，data-list-uuid字段一致，表示一个实例。
具体POST报文（fetch格式）
```
fetch("https://www.autodl.com/api/v1/instance/power_on", {
  "headers": {
    "accept": "*/*",
    "accept-language": "zh-CN,zh;q=0.9",
    "appversion": "v5.57.1",
    "authorization": "xxx",
    "cache-control": "no-cache",
    "content-type": "application/json;charset=UTF-8",
    "pragma": "no-cache",
    "priority": "u=1, i",
    "sec-ch-ua": "\"Google Chrome\";v=\"131\", \"Chromium\";v=\"131\", \"Not_A Brand\";v=\"24\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": "\"Windows\"",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "cors",
    "sec-fetch-site": "same-origin"
  },
  "referrer": "https://www.autodl.com/console/instance/list",
  "referrerPolicy": "strict-origin-when-cross-origin",
  "body": "{\"instance_uuid\":\"2dee4c9145-b0f2fdc8\"}",
  "method": "POST",
  "mode": "cors",
  "credentials": "include"
});
```
返回报文
```json
{"code":"Success","data":null,"msg":""}
```
## 关机
关机和开机的POST报文格式、返回格式基本相同，只有请求的接口不同为，`https://www.autodl.com/api/v1/instance/power_off`
## 无卡模式
无卡模式开关机请求的接口仍为`power_on`和`power_off`，区别是开机时请请求体为
```json
{"instance_uuid":"xxx","payload":"non_gpu"}
```
关机时请求体中只有`instance_uuid`字段
# 目标
- 实现登录认证功能，当token不存在或者过期后，重新登录、获取并保存token
- 获取当前实例是否空闲（是否有空闲机器）
- 使用telegram-bot-api库编写telegram bot，进行管理。实现如下功能
	- \user 设定用户名
	- \password 设定密码
	- \gpuvalid 查看所有实例是否空闲，输出机器名及其空闲gpu数量