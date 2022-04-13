

# NodeJS Example

```javascript

'use strict'

const axios = require('axios')
const CryptoJS = require('crypto-js')
let timestamp = new Date().getTime()

const paramUtils = {
    values: [],
    put(k, v) {
        let value = encodeURIComponent(v)
        this.values.push(k + "=" + value)
    },
    sortedValues() {
        return this.values.sort()
    },
    addGetParams(params) {
        Object.keys(params).forEach(k => {
            this.put(k, params[k])
        })
        this.sortedValues()
    },
    getParams(requestMethod, param) {
        if (requestMethod === 'GET') {
            this.put("signTimestamp", timestamp)
            this.addGetParams(param)
            return this.values.join("&").toString()
        } else if (requestMethod === 'POST' || requestMethod === 'PUT' || requestMethod === 'DELETE') {
            return "requestBody=" + JSON.stringify(param) + "&signTimestamp=" + timestamp
        }
    }
}

class Sign {
    constructor(method, path, param, secretKey) {
        this.method = method
        this.path = path
        this.param = param
        this.secretKey = secretKey
    }

    sign() {
        let paramValue = paramUtils.getParams(this.method, this.param)
        let payload = this.method.toUpperCase() + "\n" + this.path + "\n" + paramValue
        console.log("payload:" + payload)

        let hmacData = CryptoJS.HmacSHA256(payload, this.secretKey);
        return CryptoJS.enc.Base64.stringify(hmacData);
    }
}

// dev
// const url = 'https://dev-spot-api-gateway.poloniex.com'
// const apiKey = "IZ6TMTE8-LZFLDBCA-VPNEWXJR-RDY10005"
// const secretKey = "ae05a630947c5a8df90e6cb587f7e14406e6b5d47a22e4975ac712f6ce6e6a73fda2b5e12e145fb3a0cfab8f6c408e3015c3763cc9d2c850f2e29a9962362faa"

// sand
const url = 'https://sand-spot-api-gateway.poloniex.com'
const apiKey = "BJU56N7S-4TSNOC13-69L22ZQP-W1ME7DHQ"
const secretKey = "3cb3344bc70831207719d01036962f7237f952f3e49cadd01dbf61d1a57ca7f4e3818273dd8963116c24276f281a3d06a54fda53f81e133879678fa370d83902"

function getHeader(method, path, param) {
    const sign = new Sign(method, path, param, secretKey).sign()
    console.log(`signature:${sign}`)
    return {
        "Content-Type": "application/json",
        "key": apiKey,
        "signature": sign,
        "signTimestamp": timestamp
    }
}

function get(url, path, param = {}) {
    const headers = getHeader('GET', path, param)
    return axios.get(url + path, {params: param, headers: headers})
        .then(res => console.log(res.data))
        .catch(e => console.error(e))
}

function post(url, path, param = {}) {
    const headers = getHeader('POST', path, param)
    return axios.post(url + path, param, {headers: headers})
        .then(res => console.log(res.data))
        .catch(e => console.error(e))
}

const orderData2 = {
    "symbol": "link_usdt",
    "accountType": "spot",
    "type": "limit",
    "side": "buy",
    "timeInForce": "GTC",
    "price": "1.1",
    "amount": "1",
    "quantity": "1",
    "clientOrderId": "",
}

// Place Order
post(url, '/v2/orders', orderData2)

// getOrders
get(url, '/v2/orders', {limit: 10})

```