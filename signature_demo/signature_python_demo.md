# Python Example

```

 # -*- coding:utf-8 -*-
 import hashlib
 import urllib
 import urllib.parse
 import urllib.request
 import requests
 import time
 import hmac
 import base64
 import json
 
 
 class SDK:
 
     def __init__(self, access_key, secret_key):
         self.__access_key = access_key
         self.__secret_key = secret_key
         self.__time = int(time.time() * 1000)
 
     def __create_sign(self, params, method, path):
         timestamp = self.__time
         if method.upper() == "GET":
             params.update({"signTimestamp": timestamp})
             sorted_params = sorted(params.items(), key=lambda d: d[0], reverse=False)
             encode_params = urllib.parse.urlencode(sorted_params)
             del params["signTimestamp"]
         elif method.upper() == "POST" or "PUT" or "DELETE":
             requestBody = json.dumps(params)
             encode_params = "requestBody={}&signTimestamp={}".format(
                 requestBody, timestamp
             )
         sign_params_first = [method.upper(), path, encode_params]
         sign_params_second = "\n".join(sign_params_first)
         sign_params = sign_params_second.encode(encoding="UTF8")
         secret_key = self.__secret_key.encode(encoding="UTF8")
         digest = hmac.new(secret_key, sign_params, digestmod=hashlib.sha256).digest()
         signature = base64.b64encode(digest)
         signature = signature.decode()
         return signature
 
     def sign_req(self, host, path, method, params, headers):
         sign = self.__create_sign(params=params, method=method, path=path)
         headers.update(
             {
                 "key": self.__access_key,
                 "signTimestamp": str(self.__time),
                 "signature": sign,
             }
         )
 
         if method.upper() == "POST":
             host = "{host}{path}".format(host=host, path=path)
             response = requests.post(host, data=json.dumps(params), headers=headers)
             return response.json()
 
         if method.upper() == "GET":
             params = urllib.parse.urlencode(params)
             if params == "":
                 host = "{host}{path}".format(host=host, path=path)
             else:
                 host = "{host}{path}?{params}".format(host=host, path=path, params=params)
             response = requests.get(host, params={}, headers=headers)
             return response.json()
 
         if method.upper() == "PUT":
             host = "{host}{path}".format(host=host, path=path)
             response = requests.put(host, data=json.dumps(params), headers=headers)
             return response.json()
 
         if method.upper() == "DELETE":
             host = "{host}{path}".format(host=host, path=path)
             response = requests.delete(host, data=json.dumps(params), headers=headers)
             return response.json()
 
 if __name__ == "__main__":
     headers = {"Content-Type": "application/json"}
     host = ""    
     access_key = "access_key"    
     secret_key = "secret_key"    
     service = SDK(access_key, secret_key)
 
     # Example1: get order        
     path_req = "/v2/orders/"    
     method_req = "get"    
     params_req = {"limit": 10}
     res = service.sign_req(
         host,
         path_req,
         method_req,
         params_req,
         headers)
     print("\033[1;31m latency\033[0m", res)
 
     # Example2: place order        
     path_req = "/v2/orders"    
     method_req = "post"    
     params_req = {
         "symbol": "link_usdt",
         "accountType": "spot",
         "type": "limit",
         "side": "sell",
         "timeInForce": "GTC",
         "price": "1000",
         "amount": "1",
         "quantity": "10",
         "clientOrderId": "",
     }
     res = service.sign_req(
         host,
         path_req,
         method_req,
         params_req,
         headers)
     print("\033[1;31m latency\033[0m", res)
 
     # Example3: cancel order        
     path_req = "/v2/orders/cancelByIds"    
     method_req = "DELETE"    
     params_req = {
         "orderIds": ["29222978772373504"],
         "clientOrderIds": []
     }
     res = service.sign_req(
         host,
         path_req,
         method_req,
         params_req,
         headers)
     print("\033[1;31m latency\033[0m", res)

```