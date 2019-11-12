---
title:  nessus与awvs API
tags: [扫描器,工具,自动化]
categories:  漏扫
date: 2019-08-2 14:20:00
 
---


![nessus.jpg](http://upload-images.jianshu.io/upload_images/3941016-beb5143ed978f26e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#  前言
 最近在做自动化安全扫描器开发，在项目中想引用比较知名的扫描器，如是整理了一下API作为参考


<!--more-->
#  <font color=#C71585 size=4 face="黑体"> Nessus 6.x版本</font>
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2017/8/21 下午4:08
# @Author  : Yu BenLiu
# @Site    : QVQ
# @File    : nessus_api_6.py.py
# @Software: PyCharm
import sys
# sys.path.append("..")

import requests, json, csv, os, time
from requests.packages.urllib3.exceptions import InsecureRequestWarning

requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

# from core.settings import talscan_config,redis_task
from host_type_check import *

talscan_config = {"report_filters": {
    "awvs_white_list": ["orange", "red", "blue"],  # green,blue,orange,red四种级别
    "nessus_white_list": ["High", "Medium", "Low"],
    "bug_black_list": [  # 漏洞黑名单，过滤掉一些危害等级高，但没什么卵用的洞
        "User credentials are sent in clear text"
    ]
}
}


class Work(object):
    def __init__(self, scan_id="", scan_uuid="",scan_target="", scan_type="", scan_args="", back_fn=None):
        self.api_url = 'https://192.168.29.198:8834'
        self.username = "root"
        self.password = "ybl8651073"
        self.filter = talscan_config["report_filters"]
        self.report_save_dir = '/tmp/'
        self.verify = False
        self.token = ''
        self.enable = True
        self.scan_uuid=scan_uuid
        self.scan_id = scan_id
        self.target = scan_target
        self.scan_type = scan_type
        self.args = scan_args
        self.back_fn = back_fn
        self.result = {}

    def connect(self, method, resource, data=None, params=None):
        headers = {'X-Cookie': 'token={0}'.format(self.token), 'content-type': 'application/json'}
        data = json.dumps(data)

        try:
            if method == 'POST':
                r = requests.post(str(self.api_url + resource), data=data, headers=headers, verify=self.verify)

            elif method == 'PUT':
                r = requests.put(str(self.api_url + resource), data=data, headers=headers, verify=self.verify)
            elif method == 'DELETE':
                r = requests.delete(str(self.api_url + resource), data=data, headers=headers, verify=self.verify)
            else:
                r = requests.get(str(self.api_url + resource), params=params, headers=headers, verify=self.verify)
        except Exception, e:
            print  e
            return {"status": 3}

        if r.status_code == 200:
            try:
                data = r.json()
            except:
                data = r.content

            result = {"status": 1, "data": data}
            return result
        else:
            result = {"status": 3}
            return result

    def nessus_login(self):
        login = {'username': self.username, 'password': self.password}
        data = self.connect('POST', '/session', data=login)
        print data
        status = data["status"]
        print status
        if status == 1:
            result = {"status": 1, "data": data["data"]['token']}
            print  result
            return result
        else:
            result = {"status": 0}
            print result
            return result

    def nessus_process_status(self, sid):
        # canceled,running,completed
        data = self.nessus_login()
        status = data["status"]
        if status == 1:
            token = data["data"]
        else:
            result = {"status": 0}
            return result

        headers = {'X-Cookie': 'token={0}'.format(token), 'content-type': 'application/json'}
        url = self.api_url + '/scans/'
        data = requests.get(url=url, params=None, headers=headers, verify=self.verify)
        res = data.json()
        info1=res['scans']
        print info1
        try:
            return  info1
        except:
            return {"status": 0}
    def  nessus_policies(self,):
        data = self.nessus_login()
        status = data["status"]
        if status == 1:
            token = data["data"]
        else:
            result = {"status": 0}
            return result

        headers = {'X-Cookie': 'token={0}'.format(token), 'content-type': 'application/json'}
        url = self.api_url + '/policies/'
        data = requests.get(url=url, params=None, headers=headers, verify=self.verify)
        res = data.json()
        info=res['policies']
        print info
        try:
            return info
        except:
            return {"status": 0}


    def nessus_add_task(self):
        if is_domain(self.target) or is_host(self.target):
            create_data = {
                "uuid": self.scan_uuid,
                # "uuid": 'ad629e16-03b6-8c1d-cef6-ef8c9dd3c658d24bd260ef5f9e66',#选择策略
                "settings": {
                    "name": self.scan_id,
                    "scanner_id": "1",
                    "text_targets": self.target,
                    "enabled": False,
                    "launch_now": True,
                }
            }
            post_data = json.dumps(create_data)

            data = self.nessus_login()
            status = data["status"]
            if status == 1:
                token = data["data"]
            else:
                print  '99'
                return {"status": 2, "data": "NESSUS >>>> :登陆失败"}

            headers = {'X-Cookie': 'token={0}'.format(token), 'content-type': 'application/json'}
            r = requests.post(url=str(self.api_url + '/scans'), data=post_data, headers=headers, verify=self.verify)

            if r.status_code == 200:
                try:
                    get_id = r.json()
                except:
                    get_id = r.content

                sid = get_id['scan']['id']
                result = {"status": 1, "data": sid}
                return result
            else:
                result = {"status": 2, "data": "NESSUS >>>> :增加任务失败"}
                return result
        else:
            return {"status": 2, "data": "NESSUS >>>> :格式错误"}

    def nessus_stop_task(self, sid):
        data = self.nessus_login()
        status = data["status"]
        if status == 1:
            token = data["data"]
        else:
            result = {"status": 0}
            return result

        headers = {'X-Cookie': 'token={0}'.format(token), 'content-type': 'application/json'}
        url = str(self.api_url + '/scans/{0}/stop/'.format(sid))
        r = requests.post(url=url, params=None, headers=headers, verify=self.verify)
        if r.status_code == 200:
            result = {"status": 1}
            return result
        else:
            result = {"status": 0}
            return result

    def nessus_report_task(self, taskid, sid):
        bug_list = []

        data = self.nessus_login()
        status = data["status"]
        if status == 1:
            token = data["data"]
        else:
            result = {"status": 0}
            return result

        headers = {'X-Cookie': 'token={0}'.format(token), 'content-type': 'application/json'}
        url = str(self.api_url + '/scans/{0}/export'.format(sid))
        data = json.dumps({"format": "csv"})
        r = requests.post(url=url, data=data, headers=headers, verify=self.verify)
        try:
            file = r.json()['token']
        except:
            result = {"status": 0}
            return result
        down_file_url = str(self.api_url + '/scans/exports/{0}/download'.format(file))
        r = requests.get(url=down_file_url, headers=headers, verify=self.verify)

        csv_file = str(self.report_save_dir + "{0}_nessus.csv".format(str(taskid)))
        f = open(csv_file, 'wb')
        data = r.content
        f.write(data)
        f.close()

        csv_open_file = open(csv_file, 'rb')
        csvReader = csv.reader(csv_open_file)
        for row in csvReader:
            parameterStr = ','.join(row)
            parameters = parameterStr.split(',')
            PID = parameters[0]
            CVE = parameters[1]
            CVSS = parameters[2]
            Risk = parameters[3]
            Host = parameters[4]
            Protocol = parameters[5]
            Port = parameters[6]
            Name = parameters[7]
            Synopsis = parameters[8]
            Description = parameters[9]
            Solution = parameters[10]
            See_Also = parameters[11]
            Plugin_Output = parameters[12]

            bug_name = str(Name)
            bug_level = str(Risk)
            bug_summary = str(Synopsis) + "\r\n" + str(Description)
            bug_detail = "Bug Port : " + str(Port) + "\r\n" + "CVE : " + str(CVE)
            bug_repair = str(Solution) + "\r\n" + str(Plugin_Output)

            if str(Risk) in self.filter['nessus_white_list']:
                bug_list.append(
                    {'bug_name': bug_name, 'bug_level': bug_level, 'bug_summary': bug_summary, 'bug_detail': bug_detail,
                     'bug_repair': bug_repair})

        csv_open_file.close()
        # os.remove(csv_file)

        if len(bug_list) > 0:
            result = {"status": 1, "data": bug_list}
            return bug_list
        else:
            result = {"status": 0}
            return result

    def run(self):
        result = self.nessus_add_task()
        status = result["status"]

        if status == 1:
            nessus_id = int(result["data"])
            while True:
                time.sleep(5)
                nessus_process = self.nessus_process_status(nessus_id)
                nessus_status = nessus_process["status"]
                if nessus_status == 1:
                    nessus_process_data = nessus_process["data"].encode("utf8")
                    if nessus_process_data == "completed":
                        break

            time.sleep(20)  # 推迟20秒，获取报告;nessus任务进程到100有部分延迟结束时间
            nessus_report = self.nessus_report_task(self.scan_id, nessus_id)
            nessus_report_status = nessus_report["status"]
            if nessus_report_status == 1:
                nessus_report_data = nessus_report["data"]
                data = []
                for line in nessus_report_data:
                    task_result = {
                        "scan_id": self.scan_id,
                        "model": "nessus",
                        "bug_author": "bing",
                        "bug_name": line["bug_name"],
                        "bug_level": line["bug_level"],
                        "bug_summary": line["bug_summary"],
                        "bug_detail": line["bug_detail"],
                        "bug_repair": line["bug_repair"]
                    }
                    redis_task.sadd("nessus_result", task_result)
                    print task_result

                # 任务最终结束
                final_result = {"status": 1, "scan_id": self.scan_id, "model": "nessus"}

                redis_task.sadd("nessus_result", final_result)
                print final_result

            else:
                # 任务最终结束
                final_result = {"status": 1, "scan_id": self.scan_id, "model": "nessus"}

                redis_task.sadd("nessus_result", final_result)
                print final_result

        elif status == 2:
            nessus_error = result["data"]
            final_result = {"status": 2, "data": nessus_error, "scan_id": self.scan_id, "model": "nessus"}

            redis_task.sadd("nessus_result", final_result)
            print final_result


#t = Work("1234455","ad629e16-03b6-8c1d-cef6-ef8c9dd3c658d24bd260ef5f9e66","www.baidu.com")
#s = t.nessus_report_task(5,84)
#print s

```



#  <font color=#C71585 size=4 face="黑体"> Awvs 最新的api</font>

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Time    : 2017/8/22 下午3:08
# @Author  : Yu BenLiu
# @Site    : QVQ
# @File    : awvs_api_11.py.py
# @Software: PyCharm

import json
import requests
import requests.packages.urllib3

'''
import requests.packages.urllib3.util.ssl_
requests.packages.urllib3.util.ssl_.DEFAULT_CIPHERS = 'ALL'
or
pip install requests[security]
'''
requests.packages.urllib3.disable_warnings()
tarurl = "https://192.168.29.207:3443/"
apikey = "1986ad8c0a5b3df4d7028d5f3c06e936c38438d15a1fb4f4588e034d3c3490fef"
headers = {"X-Auth": apikey, "content-type": "application/json"}


def addtask(url=''):
    # 添加任务
    data = {"address": url, "description": url, "criticality": "10"}
    try:
        response = requests.post(tarurl + "/api/v1/targets", data=json.dumps(data), headers=headers, timeout=30,
                                 verify=False)
        result = json.loads(response.content)
        return result['target_id']
    except Exception as e:
        print(str(e))
        return


def startscan(url):
    # 先获取全部的任务.避免重复
    # 添加任务获取target_id
    # 开始扫描
    '''
    11111111-1111-1111-1111-111111111112    High Risk Vulnerabilities
    11111111-1111-1111-1111-111111111115    Weak Passwords
    11111111-1111-1111-1111-111111111117    Crawl Only
    11111111-1111-1111-1111-111111111116    Cross-site Scripting Vulnerabilities
    11111111-1111-1111-1111-111111111113    SQL Injection Vulnerabilities
    11111111-1111-1111-1111-111111111118    quick_profile_2 0   {"wvs": {"profile": "continuous_quick"}}
    11111111-1111-1111-1111-111111111114    quick_profile_1 0   {"wvs": {"profile": "continuous_full"}}
    11111111-1111-1111-1111-111111111111    Full Scan   1   {"wvs": {"profile": "Default"}}
    '''
    targets = getscan()
    if url in targets:
        return "repeat"
    else:
        target_id = addtask(url)
        data = {"target_id": target_id, "profile_id": "11111111-1111-1111-1111-111111111111",
                "schedule": {"disable": False, "start_date": None, "time_sensitive": False}}
        try:
            response = requests.post(tarurl + "/api/v1/scans", data=json.dumps(data), headers=headers, timeout=30,
                                     verify=False)
            result = json.loads(response.content)
            return result['target_id']
        except Exception as e:
            print(str(e))
            return


def getstatus(scan_id):
    # 获取scan_id的扫描状况
    try:
        response = requests.get(tarurl + "/api/v1/scans/" + str(scan_id), headers=headers, timeout=30, verify=False)
        result = json.loads(response.content)
        status = result['current_session']['status']
        # 如果是completed 表示结束.可以生成报告
        if status == "completed":
            return getreports(scan_id)
        else:
            return result['current_session']['status']
    except Exception as e:
        print(str(e))
        return


def delete_scan(scan_id):
    # 删除scan_id的扫描
    try:
        response = requests.delete(tarurl + "/api/v1/scans/" + str(scan_id), headers=headers, timeout=30, verify=False)
        # 如果是204 表示删除成功
        if response.status_code == "204":
            return True
        else:
            return False
    except Exception as e:
        print(str(e))
        return


def delete_target(scan_id):
    # 删除scan_id的扫描
    try:
        response = requests.delete(tarurl + "/api/v1/targets/" + str(scan_id), headers=headers, timeout=30,
                                   verify=False)
        # 如果是204 表示删除成功
        if response.status_code == "204":
            return True
        else:
            return False
    except Exception as e:
        print(str(e))
        return


def stop_scan(scan_id):
    # 停止scan_id的扫描
    try:
        response = requests.post(tarurl + "/api/v1/scans/" + str(scan_id + "/abort"), headers=headers, timeout=30,
                                 verify=False)
        # 如果是204 表示停止成功
        if response.status_code == "204":
            return True
        else:
            return False
    except Exception as e:
        print(str(e))
        return
def scan_status():
    # 停止scan_id的扫描
    try:
        response = requests.get(tarurl + "/api/v1/me/stats", headers=headers, timeout=30, verify=False)
        result = json.loads(response.content)
        print result
        return result
    except Exception as e:
        print(str(e))
        return


def getreports(scan_id):
    # 获取scan_id的扫描报告
    '''
    11111111-1111-1111-1111-111111111111    Developer
    21111111-1111-1111-1111-111111111111    XML
    11111111-1111-1111-1111-111111111119    OWASP Top 10 2013
    11111111-1111-1111-1111-111111111112    Quick
    '''
    data = {"template_id": "11111111-1111-1111-1111-111111111111",
            "source": {"list_type": "scans", "id_list": [scan_id]}}
    try:
        response = requests.post(tarurl + "/api/v1/reports", data=json.dumps(data), headers=headers, timeout=30,
                                 verify=False)
        result = response.headers
        report = result['Location'].replace('/api/v1/reports/', '/reports/download/')
        return tarurl.rstrip('/') + report
    except Exception as e:
        print(str(e))
        return
    finally:
        delete_scan(scan_id)


def config(url):
    target_id = addtask(url)
    # 获取全部的扫描状态
    data = {
        "excluded_paths": ["manager", "phpmyadmin", "testphp"],
        "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/59.0.3071.115 Safari/537.36",
        "custom_headers": ["Accept: */*", "Referer:" + url, "Connection: Keep-alive"],
        "custom_cookies": [{"url": url,
                            "cookie": "UM_distinctid=15da1bb9287f05-022f43184eb5d5-30667808-fa000-15da1bb9288ba9; PHPSESSID=dj9vq5fso96hpbgkdd7ok9gc83"}],
        "scan_speed": "moderate",  # sequential/slow/moderate/fast more and more fast
        "technologies": ["PHP"],  # ASP,ASP.NET,PHP,Perl,Java/J2EE,ColdFusion/Jrun,Python,Rails,FrontPage,Node.js
        # 代理
        "proxy": {
            "enabled": False,
            "address": "127.0.0.1",
            "protocol": "http",
            "port": 8080,
            "username": "aaa",
            "password": "bbb"
        },
        # 无验证码登录
        "login": {
            "kind": "automatic",
            "credentials": {
                "enabled": False,
                "username": "test",
                "password": "test"
            }
        },
        # 401认证
        "authentication": {
            "enabled": False,
            "username": "test",
            "password": "test"
        }
    }
    try:
        res = requests.patch(tarurl + "/api/v1/targets/" + str(target_id) + "/configuration", data=json.dumps(data),
                             headers=headers, timeout=30 * 4, verify=False)

        data = {"target_id": target_id, "profile_id": "11111111-1111-1111-1111-111111111111",
                "schedule": {"disable": False, "start_date": None, "time_sensitive": False}}
        try:
            response = requests.post(tarurl + "/api/v1/scans", data=json.dumps(data), headers=headers, timeout=30,
                                     verify=False)
            result = json.loads(response.content)
            return result['target_id']
        except Exception as e:
            print(str(e))
            return
    except Exception as e:
        raise e

def getvulnerabilities():
    # 停止scan_id的扫描
    try:
        response = requests.get(tarurl + "/api/v1/vulnerabilities", headers=headers, timeout=30, verify=False)
        result = json.loads(response.content)
        print result
        return result
    except Exception as e:
        print(str(e))
        return
def getscan():
    # 获取全部的扫描状态
    targets = []
    try:
        response = requests.get(tarurl + "/api/v1/scans", headers=headers, timeout=30, verify=False)
        results = json.loads(response.content)
        return results
    except Exception as e:
        raise e
def getvulnerabilitiesinfo(sid):
    # 停止scan_id的扫描
    try:
        response = requests.get(tarurl + "/api/v1/vulnerabilities/"+sid, headers=headers, timeout=30, verify=False)
        result = json.loads(response.content)
        print result
        return result
    except Exception as e:
        print(str(e))
        return


if __name__ == '__main__':
    info=getscan()
    print info
    print type(info)

    #print getreports('f22d4aa1-e2de-4307-bd9d-ddf3aa531bc1',locals())
   # print config('http://testhtml5.vulnweb.com/')



```




