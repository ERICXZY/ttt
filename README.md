import requests
import time
import pymongo
import json
import datetime
import ipdb
city = 'beijing'
source = 'room107'
ori_coll = '%s_rental' % source
room107_phone_coll = '%s_landlord_phone' % source
MONGODB_URI = 'mongodb://52.80.143.54:27018/'
mongo_client = pymongo.MongoClient(MONGODB_URI)


def get_now():
    return int(time.time()*1000)


def get_data_from_mongodb():
    ori_collection = mongo_client[city][ori_coll]
    ret = set()
    select_cond = {"brokerId": {'$exists': True}}
    select_field = {'brokerId': 1}
    for doc in ori_collection.find(select_cond, select_field):
        ret.add(doc['brokerId'])
    return ret

def get_lanlordId_from_mongodb():
    lanlordId_set = set()
    lanlordId_collection = mongo_client[city][room107_phone_coll]
    select_cond = {'lanlordId': {'$exists': True}}
    select_field = {'lanlordId': 1}
    for doc in lanlordId_collection.find(select_cond, select_field):
        lanlordId_set.add(doc['lanlordId'])
    # ipdb.set_trace()
    return lanlordId_set

def insert_room107_phone(brokerid, result):
    print("insert %s data into beijing %s" % (brokerid, room107_phone_coll))
    phone_collection = mongo_client[city][room107_phone_coll]
    doc = {}
    doc['lanlordId'] = brokerid
    doc['lanlordPhone'] = result.get('telephone', '').replace('-', '')
    if result.get('wechat', ''):
        doc['landlordWechat'] = result.get('wechat', '')
    if result.get('qq', ''):
        doc['landlordQQ'] = result.get('qq', '')
    doc['insert_date'] = datetime.datetime.now()
    return phone_collection.insert_one(doc).inserted_id

# {"authStatus":0}
# {"contact":{"telephone":"185-0101-5206","qq":"","wechat":"377988030","status":true},"authStatus":2}
def run_task():
    brokerid_set = get_data_from_mongodb()
    lanlordId_set = get_lanlordId_from_mongodb()
    brokerid_set.difference_update(lanlordId_set)
    brokerid_set.difference_update(lanlordId_set)
    for brokerId in brokerid_set:
        index = 0
        login_list = [
            {'username': '18813105413', 'password': 'qwertyuiop', 'errorCode': ''},
            {'username': '18835105356', 'password': 'tan1003939032', 'errorCode': ''}
        ]
        s = requests.session()
        headers = {
            'Host': 'www.107room.com',
            'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36',
            'Connection': 'keep-alive',
            'Accept-Language': 'en-US,en;q=0.8',
        }
        while True:
            if index < len(login_list):
                time.sleep(20)
                data = login_list[index]
                s.post('http://www.107room.com/user/login/submit', data=data, headers=headers, timeout=30)
                time.sleep(20)
                response = s.get('http://www.107room.com/house/apply/%s?date=%s' % (brokerId, get_now()))
                result = json.loads(response.text)
                if result['contact']['status'] == False:
                    index += 1
                    print(index)
                    continue
                else:
                    r = insert_room107_phone(brokerId, result['contact'])
                    print(r)
                    if r:
                        break
            else:
                break
        if index >= len(login_list):
            break

if __name__ == '__main__':
    run_task()
