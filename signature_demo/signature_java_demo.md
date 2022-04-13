
###  Dependencies

```xml
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>4.5.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.7</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.76</version>
        </dependency>

        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>31.0.1-jre</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.20</version>
        </dependency>
```


### Test Main Class

```java

import com.google.common.collect.Lists;

public class MainTest {

    private final static String API_KEY = "api_key";
    private final static String SECRET_KEY = "secret_key";
    private final static String HOST = "https://sandbox-api.poloniex.com";

    public static void main(String[] args) throws InterruptedException {

        PoloniexApiClient poloniexApiClient = new PoloniexApiClient(API_KEY, SECRET_KEY, HOST);


        Long buyOrderId = poloniexApiClient.placeOrder("ETH_USDT", "SPOT", "LIMIT", "BUY", "GTC", "3360", "1", "T_D_UP_" + System.currentTimeMillis());
        Long sellOrderId = poloniexApiClient.placeOrder("ETH_USDT", "SPOT", "LIMIT", "SELL", "GTC", "3380", "1", "T_D_UP_" + System.currentTimeMillis());

        poloniexApiClient.cancelByIds(Lists.newArrayList(buyOrderId, sellOrderId), null);

        poloniexApiClient.batchCancelAllOrders(null, "ETH_USDT");

    }

}
```

### SDK Demo Client

```java

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONArray;
import com.alibaba.fastjson.JSONObject;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import okhttp3.Request;
import org.apache.commons.lang3.StringUtils;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

@Data
@Builder
@AllArgsConstructor
@Slf4j
public class PoloniexApiClient {

    private final static String V2_ACCOUNTS = "/v2/accounts/";
    private final static String V2_ORDERS = "/v2/orders";
    private final static String V2_ORDERS_CANCELBYIDS = "/v2/orders/cancelByIds";
    private final static String V2_ORDERS_BATCH_CANCEL_ALL_ORDERS = "/v2/orders/batchCancelAllOrders";

    private final String apiKey;

    private final String secretKey;

    private final String host;

    public List<PoloniexOrder> getOrders(String symbol) {
        try {
            Map<String, Object> map = new HashMap<>();
            if (StringUtils.isNotBlank(symbol)) {
                map.put("symbol", symbol);
            }
            Request request = PoloniexSignatureHelper.generateGetRequest(apiKey, secretKey, host, V2_ORDERS, map);
            String response = PoloniexSignatureHelper.executeRequest(request);
            log.info("getOrdersResponse: {}", response);

            List<PoloniexOrder> orderList = JSONArray.parseArray(response, PoloniexOrder.class);
            return orderList;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    public void accounts() {
        try {
            Map<String, Object> map = new HashMap<String, Object>();
            Request request = PoloniexSignatureHelper.generateGetRequest(apiKey, secretKey, host, V2_ACCOUNTS, map);
            String response = PoloniexSignatureHelper.executeRequest(request);
            log.info("accountResponse: {}", response);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public Long placeOrder(String symbol, String accountType, String type, String side, String timeInForce, String price, String quantity, String clientOrderId) {

        Map<String, Object> paramMap = new HashMap<String, Object>();
        paramMap.put("symbol", symbol);
        paramMap.put("accountType", accountType);
        paramMap.put("type", type);
        paramMap.put("side", side);
        paramMap.put("timeInForce", timeInForce);
        paramMap.put("price", price);
        paramMap.put("quantity", quantity);
        if (StringUtils.isNotBlank(clientOrderId)) {
            paramMap.put("clientOrderId", clientOrderId);
        }

        Request request = PoloniexSignatureHelper.generateBodyRequest(apiKey, secretKey, host, PoloniexSignatureHelper.REQUEST_METHOD_POST, V2_ORDERS, paramMap);
        String response = PoloniexSignatureHelper.executeRequest(request);
        log.info("placeResponse: {}  request:{} ", response, JSON.toJSONString(paramMap));
        JSONObject responseObj = JSON.parseObject(response);

        String orderId = responseObj.getString("id");
        return Long.valueOf(orderId);
    }

    public void cancelByIds(List<Long> orderIds, List<String> clientOrderIds) {
        Map<String, Object> paramMap = new HashMap<String, Object>();

        if (Objects.nonNull(orderIds) && orderIds.size() > 0) {
            paramMap.put("orderIds", orderIds);
        }

        if (Objects.nonNull(clientOrderIds) && clientOrderIds.size() > 0) {
            paramMap.put("clientOrderIds", clientOrderIds);
        }


        Request request = PoloniexSignatureHelper.generateBodyRequest(apiKey, secretKey, host, PoloniexSignatureHelper.REQUEST_METHOD_DELETE, V2_ORDERS_CANCELBYIDS, paramMap);
        String response = PoloniexSignatureHelper.executeRequest(request);
        log.info("cancelResponse: {}   \n request:{}", response, JSON.toJSONString(paramMap));
    }


    public void batchCancelAllOrders(String accountType, String symbol) {

        Map<String, Object> paramMap = new HashMap<String, Object>();
        if (StringUtils.isNotBlank(symbol)) {
            paramMap.put("symbol", symbol);
        }

        if (StringUtils.isNotBlank(accountType)) {
            paramMap.put("accountType", accountType);
        }
        Request request = PoloniexSignatureHelper.generateBodyRequest(apiKey, secretKey, host, PoloniexSignatureHelper.REQUEST_METHOD_DELETE, V2_ORDERS_BATCH_CANCEL_ALL_ORDERS, paramMap);
        String response = PoloniexSignatureHelper.executeRequest(request);
        log.info("batchCancelAllOrder:{} \n request:{}", response, JSON.toJSONString(paramMap));

    }

    
}

```

### Signature Helper Class

```java

import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import okhttp3.*;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.nio.charset.StandardCharsets;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.Base64;
import java.util.Map;
import java.util.Objects;
import java.util.TreeMap;

@Slf4j
public class PoloniexSignatureHelper {

    //    static OkHttpClient okHttpClient = new OkHttpClient();
    static OkHttpClient okHttpClient = new OkHttpClient.Builder()
            .addNetworkInterceptor(new Interceptor() {
                @Override
                public Response intercept(Chain chain) throws IOException {

                    Request request = chain.request();

                    String path = request.url().toString();

                    long startTs = System.currentTimeMillis();
                    Response response = chain.proceed(request);
                    log.info("requestLatency: {} - {} , latency:{}", request.method(), path, (System.currentTimeMillis() - startTs));
                    return response;
                }
            })

            .build();


    public static final MediaType JSON_TYPE = MediaType.parse("application/json");

    public final static String REQUEST_METHOD_GET = "GET";
    public final static String REQUEST_METHOD_POST = "POST";
    public final static String REQUEST_METHOD_DELETE = "DELETE";

    public final static String HEADER_TIMESTAMP = "signTimestamp";
    public final static String HEADER_KEY = "key";
    public final static String HEADER_SIGN_METHOD = "signMethod";
    public final static String HEADER_SIGN_VERSION = "signVersion";
    public final static String HEADER_SIGNATURE = "signature";

    public static final String SIGNATURE_METHOD_VALUE = "HmacSHA256";
    public static final String SIGNATURE_VERSION_VALUE = "2";

    public static Request generateGetRequest(String apiKey, String secretKey, String host, String path, Map<String, Object> paramMap) throws UnsupportedEncodingException {
        String timestamp = String.valueOf(System.currentTimeMillis());
        String payload = generateSignaturePayload(REQUEST_METHOD_GET, path, timestamp, paramMap);

        System.out.println("==========GET payload string=============");

        System.out.println(payload);
        System.out.println("==========GET payload string=============");
        String signature = generateSignature(secretKey, payload);

        String url = host + path;

        if (Objects.nonNull(paramMap) && paramMap.size() > 0) {
            StringBuilder queryStringBuffer = new StringBuilder("?");
            for (Map.Entry<String, Object> entry : paramMap.entrySet()) {
                queryStringBuffer.append(entry.getKey()).append("=").append(entry.getValue()).append("&");
            }

            String queryString = queryStringBuffer.substring(0, queryStringBuffer.length() - 1);
            url = url + queryString;
        }

        Request request = new Request.Builder()
                .url(url)
                .addHeader(HEADER_KEY, apiKey)
                .addHeader(HEADER_SIGN_METHOD, SIGNATURE_METHOD_VALUE)
                .addHeader(HEADER_SIGN_VERSION, SIGNATURE_VERSION_VALUE)
                .addHeader(HEADER_TIMESTAMP, timestamp)
                .addHeader(HEADER_SIGNATURE, signature)
                .build();

        return request;
    }


    public static Request generateBodyRequest(String apiKey, String secretKey, String host, String method, String path, Map<String, Object> paramMap) {
        String timestamp = String.valueOf(System.currentTimeMillis());
        String payload = generatePostSignaturePayload(method, path, timestamp, paramMap);

        System.out.println("==========BODY payload string=============");

        System.out.println(payload);
        System.out.println("==========BODY payload string=============");
        String signature = generateSignature(secretKey, payload);
        System.out.println("signature: " + signature);

        String url = host + path;

        Request request = new Request.Builder()
                .url(url)
                .addHeader(HEADER_KEY, apiKey)
                .addHeader(HEADER_SIGN_METHOD, SIGNATURE_METHOD_VALUE)
                .addHeader(HEADER_SIGN_VERSION, SIGNATURE_VERSION_VALUE)
                .addHeader(HEADER_TIMESTAMP, timestamp)
                .addHeader(HEADER_SIGNATURE, signature)
                .build();

        if (REQUEST_METHOD_POST.equalsIgnoreCase(method)) {
            request = request.newBuilder().post(RequestBody.create(JSON_TYPE, JSON.toJSONString(paramMap))).build();
        } else if (REQUEST_METHOD_DELETE.equalsIgnoreCase(method)) {
            request = request.newBuilder().delete(RequestBody.create(JSON_TYPE, JSON.toJSONString(paramMap))).build();
        }

        return request;
    }


    public static String executeRequest(Request request) {
        try {
            Response response = okHttpClient.newCall(request).execute();
            if (response.code() == 200) {
                return response.body().string();
            } else {
                throw new RuntimeException("[RESPONSE] RESPONSE ERROR STATUS: " + response.code() + "  BODY: " + (Objects.nonNull(response.body()) ? response.body().string() : " No Body"));
            }
        } catch (IOException e) {
            e.printStackTrace();
            throw new RuntimeException("[EXECUTE] NETWORK IO EXCEPTION " + e.getMessage());
        }
    }

    public static String generateSignaturePayload(String method, String path, String timestamp, Map<String, Object> paramMap) {

        TreeMap<String, Object> sortMap = new TreeMap<>(paramMap);
        sortMap.put(HEADER_TIMESTAMP, timestamp);

        StringBuilder payloadBuffer = new StringBuilder();
        payloadBuffer.append(method).append("\n")
                .append(path).append("\n");

        for (Map.Entry<String, Object> entry : sortMap.entrySet()) {
            payloadBuffer.append(entry.getKey()).append("=").append(entry.getValue()).append("&");
        }

        String payload = payloadBuffer.substring(0, payloadBuffer.length() - 1);
        return payload;
    }

    public static String generatePostSignaturePayload(String method, String path, String timestamp, Map<String, Object> paramMap) {

        StringBuilder payloadBuffer = new StringBuilder();
        payloadBuffer.append(method).append("\n")
                .append(path).append("\n")
                .append("requestBody=").append(JSON.toJSONString(paramMap))
                .append("&").append(HEADER_TIMESTAMP).append("=").append(timestamp);


        return payloadBuffer.toString();
    }

    public static String generateSignature(String secretKey, String payload) {
        Mac hmacSha256;
        try {
            hmacSha256 = Mac.getInstance(SIGNATURE_METHOD_VALUE);
            SecretKeySpec secKey = new SecretKeySpec(secretKey.getBytes(StandardCharsets.UTF_8), SIGNATURE_METHOD_VALUE);
            hmacSha256.init(secKey);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("[Signature] No such algorithm: " + e.getMessage());
        } catch (InvalidKeyException e) {
            throw new RuntimeException("[Signature] Invalid key: " + e.getMessage());
        }
        byte[] hash = hmacSha256.doFinal(payload.getBytes(StandardCharsets.UTF_8));

        String actualSign = Base64.getEncoder().encodeToString(hash);
        return actualSign;
    }

}

```


### Model

```java

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class PoloniexOrder {

    private Long id;

    private String clientOrderId;

    private String symbol;

    private String state;

    private String accountType;

    private String exchange;

    private String side;

    private String type;

    private String timeInForce;

    private BigDecimal quantity;

    private BigDecimal price;

    private BigDecimal avgPrice;

    private BigDecimal amount;

    private BigDecimal filledQuantity;

    private BigDecimal filledAmount;

    private Long createTime;

    private Long updateTime;

}


```