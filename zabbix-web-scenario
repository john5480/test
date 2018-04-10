#!/usr/bin/env python
# coding: utf-8
# Project Source: https://github.com/renanvicente/zabbix-web-scenario
# Updated by: John5480 2018/4/10
# Version:        0.0.1
# Author:         Renan Vicente
# Mail:           renanvice@gmail.com
# Website:        http://www.renanvicente.com
# Github:         https://www.github.com/renanvicente
# Linkedin:       http://www.linkedin.com/pub/renan-silva/6a/802/59b/en

from pyzabbix import ZabbixAPI
import sys
from re import compile,IGNORECASE
reload(sys)
sys.setdefaultencoding("utf-8")
"""
This is a script to add a web scenario and create a trigger.
"""

# The hostname at which the Zabbix web interface is available


def authentication(server_url,user,password):

  if server_url and user and password:
    ZABBIX_SERVER = server_url
    zapi = ZabbixAPI(ZABBIX_SERVER)
    try:
      # Login to the Zabbix API
      zapi.login(user,password)
      return zapi
    except Exception, e:
      print(e)
      sys.exit(1)
  else:
    print('Zabbix Server url , user and password are required, try use --help')
    sys.exit(1)


def create_web_scenario(self,name,url,group,hostid,applicationid, url_name='Homepage',status='200'):
  request = ZabbixAPI.do_request(self, 'httptest.get', params={ "filter": {"name": name}})
  if request['result']:
    print('Host "%s" already registered' % name)
    sys.exit(1)
  else:
    try:
      ZabbixAPI.do_request(self, 'httptest.create',params={"name": name,"hostid": hostid,"applicationid": applicationid, "delay": '60',"retries": '3', "steps": [ { 'name': url_name, 'url': url,'status_codes': status, 'no': '1'} ] } )
      triggers = create_trigger(auth,name,url,group)
    except Exception, e:
      print(e)
      sys.exit(1)

def create_by_file(auth, group, hostid, applicationid, filename):
  try:
    file_to_parse = open(filename,'r')
    try:
      for line in file_to_parse:
        values = line.split(',')
        try:
          name = values[0]
          url = values[1]
        except IndexError, e:
          print('Need at minimun 2 params Traceback %s:' % e)
          sys.exit(1)
        try:
          url_name = values[2]
        except IndexError:
          url_name = None
        if url_name:
          create_web_scenario(auth,name,url,group,hostid,applicationid, url_name)
        else:
          create_web_scenario(auth,name,url,group,hostid, applicationid)
    finally:
      file_to_parse.close()
  except IOError:
    print('could not open the file %s' % filename)

def create_trigger(auth,name,url,group):
  triggers = auth.trigger.create(description=name,comments="The website below does not response the HTTP request ( visit website member ) at least 120 seconds, this warning means that the website is down or unstable.\n%s" % url,expression='{%s:web.test.fail[%s].sum(120)}=1' % (group,name),priority=5)
  return triggers


if __name__ == '__main__':
  from optparse import OptionParser
  parser = OptionParser()
  parser.add_option("-z","--zabbix",dest="server_url",help='URL for Zabbix Server',metavar='ZABBIX_SERVER')
  parser.add_option('-n','--name',dest='name',help='URL名称',metavar='NAME')
  parser.add_option('-w','--url-name',dest='url_name',help='测试步骤名称',metavar='URL_NAME')
  parser.add_option('--url',dest='url',help='URL地址',metavar='URL')
  parser.add_option('-s','--status',dest='status',help='Status Code',metavar='STATUS_CODE')
  parser.add_option('-u','--user',dest='user',help='User for authentication',metavar='USER')
  parser.add_option('-p','--password',dest='password',help='Password for authentication',metavar='PASSWORD')
  parser.add_option('-f','--file',dest='filename',help='File with Name,URL',metavar='FILE')
  parser.add_option('-g','--group-name',dest='group',help='主机名',metavar='GROUP')
  parser.add_option('-i','--host-id',dest='hostid',help='Host ID',metavar='HOSTID')
  parser.add_option('-a','--application-id',dest='applicationid',help='Application ID',metavar='Application ID')
  (options, args) = parser.parse_args()
  auth = authentication(options.server_url,options.user,options.password)
  if options.filename:
    create_by_file(auth, options.group, options.hostid, options.applicationid, options.filename)
  else:
    if not options.group:
      print('Group must be required')
      sys.exit(1)
    if options.status:
      if options.url_name:
        web_scenario = create_web_scenario(auth, options.name,options.url,options.group, options.hostid, options.applicationid, options.url_name,options.status)
      else:
        web_scenario = create_web_scenario(auth, options.name,None,options.url, options.group, options.hostid, options.applicationid, options.status)
    else:
      if options.url_name:
        web_scenario = create_web_scenario(auth, options.name,options.url, options.group, options.hostid, options.applicationid, options.url_name)
      else:
        web_scenario = create_web_scenario(auth, options.name,options.url, options.group, options.hostid, options.applicationid)
