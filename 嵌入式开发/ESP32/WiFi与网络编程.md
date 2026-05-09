# ESP32 WiFi与网络编程

## 📋 概述

ESP32内置Wi-Fi功能是其最重要的特性之一，支持802.11 b/g/n协议，可工作在STA（站点）、AP（热点）或STA+AP混合模式。本文将详细介绍ESP32的Wi-Fi编程和网络应用开发。

## 🌐 WiFi基础配置

### WiFi模式说明

| 模式 | 说明 | 应用场景 |
|------|------|----------|
| STA模式 | 作为客户端连接到路由器 | 一般IoT设备联网 |
| AP模式 | 作为热点供其他设备连接 | 配网、设备间通信 |
| STA+AP模式 | 同时支持客户端和热点 | 网关设备、中继器 |

### WiFi基础初始化

```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "nvs_flash.h"

static const char *TAG = "WiFi";

// WiFi事件组
static EventGroupHandle_t s_wifi_event_group;
#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

// 重连计数器
static int s_retry_num = 0;
#define ESP_MAXIMUM_RETRY  5

// WiFi配置
#define ESP_WIFI_SSID      "Your_WiFi_SSID"
#define ESP_WIFI_PASS      "Your_WiFi_Password"
#define ESP_WIFI_CHANNEL   6
#define ESP_MAX_STA_CONN   4

// WiFi事件处理器
static void event_handler(void* arg, esp_event_base_t event_base,
                         int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < ESP_MAXIMUM_RETRY) {
            esp_wifi_connect();
            s_retry_num++;
            ESP_LOGI(TAG, "retry to connect to the AP");
        } else {
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
        ESP_LOGI(TAG,"connect to the AP fail");
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "got ip:" IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(void)
{
    s_wifi_event_group = xEventGroupCreate();

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    esp_event_handler_instance_t instance_any_id;
    esp_event_handler_instance_t instance_got_ip;
    
    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT,
                                                        ESP_EVENT_ANY_ID,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_any_id));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT,
                                                        IP_EVENT_STA_GOT_IP,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_got_ip));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = ESP_WIFI_SSID,
            .password = ESP_WIFI_PASS,
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
            .pmf_cfg = {
                .capable = true,
                .required = false
            },
        },
    };
    
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "wifi_init_sta finished.");

    // 等待连接结果
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
            pdFALSE,
            pdFALSE,
            portMAX_DELAY);

    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "connected to ap SSID:%s password:%s",
                 ESP_WIFI_SSID, ESP_WIFI_PASS);
    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGI(TAG, "Failed to connect to SSID:%s, password:%s",
                 ESP_WIFI_SSID, ESP_WIFI_PASS);
    } else {
        ESP_LOGE(TAG, "UNEXPECTED EVENT");
    }
}
```

## 🔍 WiFi扫描和连接管理

### WiFi网络扫描

```c
#include "esp_wifi.h"
#include "string.h"

#define DEFAULT_SCAN_LIST_SIZE 20

void wifi_scan_example(void)
{
    wifi_scan_config_t scan_config = {
        .ssid = NULL,           // 扫描所有SSID
        .bssid = NULL,          // 扫描所有BSSID
        .channel = 0,           // 扫描所有信道
        .show_hidden = false,   // 不显示隐藏网络
        .scan_type = WIFI_SCAN_TYPE_ACTIVE,
        .scan_time = {
            .active = {
                .min = 120,     // 最小扫描时间
                .max = 150,     // 最大扫描时间
            },
        },
    };

    ESP_LOGI(TAG, "Start scanning...");
    ESP_ERROR_CHECK(esp_wifi_scan_start(&scan_config, true));

    uint16_t number = DEFAULT_SCAN_LIST_SIZE;
    wifi_ap_record_t ap_info[DEFAULT_SCAN_LIST_SIZE];
    uint16_t ap_count = 0;
    
    memset(ap_info, 0, sizeof(ap_info));
    ESP_ERROR_CHECK(esp_wifi_scan_get_ap_records(&number, ap_info));
    ESP_ERROR_CHECK(esp_wifi_scan_get_ap_num(&ap_count));
    
    ESP_LOGI(TAG, "Total APs scanned = %u", ap_count);
    
    for (int i = 0; (i < DEFAULT_SCAN_LIST_SIZE) && (i < ap_count); i++) {
        ESP_LOGI(TAG, "SSID \t\t%s", ap_info[i].ssid);
        ESP_LOGI(TAG, "RSSI \t\t%d", ap_info[i].rssi);
        ESP_LOGI(TAG, "Channel \t\t%d", ap_info[i].primary);
        
        // 打印加密类型
        switch (ap_info[i].authmode) {
            case WIFI_AUTH_OPEN:
                ESP_LOGI(TAG, "Authmode \t\tWIFI_AUTH_OPEN");
                break;
            case WIFI_AUTH_WEP:
                ESP_LOGI(TAG, "Authmode \t\tWIFI_AUTH_WEP");
                break;
            case WIFI_AUTH_WPA_PSK:
                ESP_LOGI(TAG, "Authmode \t\tWIFI_AUTH_WPA_PSK");
                break;
            case WIFI_AUTH_WPA2_PSK:
                ESP_LOGI(TAG, "Authmode \t\tWIFI_AUTH_WPA2_PSK");
                break;
            case WIFI_AUTH_WPA_WPA2_PSK:
                ESP_LOGI(TAG, "Authmode \t\tWIFI_AUTH_WPA_WPA2_PSK");
                break;
            case WIFI_AUTH_WPA3_PSK:
                ESP_LOGI(TAG, "Authmode \t\tWIFI_AUTH_WPA3_PSK");
                break;
            default:
                ESP_LOGI(TAG, "Authmode \t\tUnknown");
                break;
        }
        ESP_LOGI(TAG, "--------------------------------");
    }
}
```

### 智能WiFi连接管理

```c
#include "esp_smartconfig.h"

// 存储多个WiFi配置
typedef struct {
    char ssid[32];
    char password[64];
    wifi_auth_mode_t authmode;
    int8_t rssi_threshold;
} wifi_profile_t;

#define MAX_WIFI_PROFILES 5
static wifi_profile_t wifi_profiles[MAX_WIFI_PROFILES];
static int profile_count = 0;

// 添加WiFi配置
esp_err_t wifi_add_profile(const char* ssid, const char* password, 
                          wifi_auth_mode_t authmode, int8_t rssi_threshold)
{
    if (profile_count >= MAX_WIFI_PROFILES) {
        return ESP_ERR_NO_MEM;
    }
    
    strncpy(wifi_profiles[profile_count].ssid, ssid, sizeof(wifi_profiles[profile_count].ssid));
    strncpy(wifi_profiles[profile_count].password, password, sizeof(wifi_profiles[profile_count].password));
    wifi_profiles[profile_count].authmode = authmode;
    wifi_profiles[profile_count].rssi_threshold = rssi_threshold;
    profile_count++;
    
    return ESP_OK;
}

// 智能选择最佳WiFi网络
esp_err_t wifi_smart_connect(void)
{
    wifi_ap_record_t ap_info[DEFAULT_SCAN_LIST_SIZE];
    uint16_t number = DEFAULT_SCAN_LIST_SIZE;
    uint16_t ap_count = 0;
    
    // 扫描网络
    ESP_ERROR_CHECK(esp_wifi_scan_start(NULL, true));
    ESP_ERROR_CHECK(esp_wifi_scan_get_ap_records(&number, ap_info));
    ESP_ERROR_CHECK(esp_wifi_scan_get_ap_num(&ap_count));
    
    // 寻找最佳匹配网络
    wifi_ap_record_t* best_ap = NULL;
    wifi_profile_t* best_profile = NULL;
    int8_t best_rssi = -100;
    
    for (int i = 0; i < ap_count; i++) {
        for (int j = 0; j < profile_count; j++) {
            if (strcmp((char*)ap_info[i].ssid, wifi_profiles[j].ssid) == 0 &&
                ap_info[i].rssi > wifi_profiles[j].rssi_threshold &&
                ap_info[i].rssi > best_rssi) {
                best_ap = &ap_info[i];
                best_profile = &wifi_profiles[j];
                best_rssi = ap_info[i].rssi;
            }
        }
    }
    
    if (best_ap && best_profile) {
        ESP_LOGI(TAG, "Connecting to %s (RSSI: %d)", best_profile->ssid, best_rssi);
        
        wifi_config_t wifi_config = {0};
        strcpy((char*)wifi_config.sta.ssid, best_profile->ssid);
        strcpy((char*)wifi_config.sta.password, best_profile->password);
        wifi_config.sta.threshold.authmode = best_profile->authmode;
        
        ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
        ESP_ERROR_CHECK(esp_wifi_connect());
        
        return ESP_OK;
    }
    
    ESP_LOGW(TAG, "No suitable WiFi network found");
    return ESP_ERR_NOT_FOUND;
}
```

## 📱 AP模式和配网

### 创建WiFi热点

```c
void wifi_init_softap(void)
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_ap();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT,
                                                        ESP_EVENT_ANY_ID,
                                                        &wifi_event_handler,
                                                        NULL,
                                                        NULL));

    wifi_config_t wifi_config = {
        .ap = {
            .ssid = "ESP32_AP",
            .ssid_len = strlen("ESP32_AP"),
            .channel = ESP_WIFI_CHANNEL,
            .password = "12345678",
            .max_connection = ESP_MAX_STA_CONN,
            .authmode = WIFI_AUTH_WPA_WPA2_PSK
        },
    };
    
    if (strlen("12345678") == 0) {
        wifi_config.ap.authmode = WIFI_AUTH_OPEN;
    }

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "wifi_init_softap finished. SSID:%s password:%s channel:%d",
             "ESP32_AP", "12345678", ESP_WIFI_CHANNEL);
}

// AP模式事件处理
static void wifi_event_handler(void* arg, esp_event_base_t event_base,
                              int32_t event_id, void* event_data)
{
    if (event_id == WIFI_EVENT_AP_STACONNECTED) {
        wifi_event_ap_staconnected_t* event = (wifi_event_ap_staconnected_t*) event_data;
        ESP_LOGI(TAG, "station "MACSTR" join, AID=%d",
                 MAC2STR(event->mac), event->aid);
    } else if (event_id == WIFI_EVENT_AP_STADISCONNECTED) {
        wifi_event_ap_stadisconnected_t* event = (wifi_event_ap_stadisconnected_t*) event_data;
        ESP_LOGI(TAG, "station "MACSTR" leave, AID=%d",
                 MAC2STR(event->mac), event->aid);
    }
}
```

### 网页配网系统

```c
#include "esp_http_server.h"
#include "cJSON.h"

static const char* config_html = 
"<!DOCTYPE html>"
"<html>"
"<head><title>ESP32 WiFi Config</title></head>"
"<body>"
"<h1>WiFi Configuration</h1>"
"<form action='/configure' method='post'>"
"<label>SSID:</label><br>"
"<input type='text' name='ssid' required><br><br>"
"<label>Password:</label><br>"
"<input type='password' name='password'><br><br>"
"<input type='submit' value='Connect'>"
"</form>"
"</body>"
"</html>";

// 配网页面处理
static esp_err_t config_page_handler(httpd_req_t *req)
{
    httpd_resp_set_type(req, "text/html");
    httpd_resp_send(req, config_html, strlen(config_html));
    return ESP_OK;
}

// WiFi配置处理
static esp_err_t wifi_config_handler(httpd_req_t *req)
{
    char buf[100];
    int ret, remaining = req->content_len;

    if (remaining >= sizeof(buf)) {
        httpd_resp_send_err(req, HTTPD_400_BAD_REQUEST, "Content too long");
        return ESP_FAIL;
    }

    ret = httpd_req_recv(req, buf, remaining);
    if (ret <= 0) {
        if (ret == HTTPD_SOCK_ERR_TIMEOUT) {
            httpd_resp_send_err(req, HTTPD_408_REQ_TIMEOUT, "Request Timeout");
        }
        return ESP_FAIL;
    }

    buf[ret] = '\0';

    // 解析POST数据
    char ssid[32] = {0};
    char password[64] = {0};
    
    // 简单的URL解码和参数提取
    char* ssid_start = strstr(buf, "ssid=");
    char* pass_start = strstr(buf, "password=");
    
    if (ssid_start) {
        ssid_start += 5; // 跳过"ssid="
        char* ssid_end = strchr(ssid_start, '&');
        if (!ssid_end) ssid_end = buf + strlen(buf);
        
        int ssid_len = ssid_end - ssid_start;
        if (ssid_len < sizeof(ssid)) {
            strncpy(ssid, ssid_start, ssid_len);
            ssid[ssid_len] = '\0';
        }
    }
    
    if (pass_start) {
        pass_start += 9; // 跳过"password="
        char* pass_end = strchr(pass_start, '&');
        if (!pass_end) pass_end = buf + strlen(buf);
        
        int pass_len = pass_end - pass_start;
        if (pass_len < sizeof(password)) {
            strncpy(password, pass_start, pass_len);
            password[pass_len] = '\0';
        }
    }

    ESP_LOGI(TAG, "Received WiFi config: SSID=%s", ssid);

    // 配置WiFi连接
    wifi_config_t wifi_config = {0};
    strcpy((char*)wifi_config.sta.ssid, ssid);
    strcpy((char*)wifi_config.sta.password, password);
    
    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(WIFI_IF_STA, &wifi_config);
    esp_wifi_connect();

    // 发送成功响应
    const char* resp_str = "WiFi configuration saved. Attempting to connect...";
    httpd_resp_send(req, resp_str, strlen(resp_str));
    
    return ESP_OK;
}

// 启动配网HTTP服务器
static httpd_handle_t start_webserver(void)
{
    httpd_handle_t server = NULL;
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    config.lru_purge_enable = true;

    ESP_LOGI(TAG, "Starting server on port: '%d'", config.server_port);
    if (httpd_start(&server, &config) == ESP_OK) {
        ESP_LOGI(TAG, "Registering URI handlers");
        
        httpd_uri_t config_page = {
            .uri       = "/",
            .method    = HTTP_GET,
            .handler   = config_page_handler,
            .user_ctx  = NULL
        };
        httpd_register_uri_handler(server, &config_page);

        httpd_uri_t wifi_config = {
            .uri       = "/configure",
            .method    = HTTP_POST,
            .handler   = wifi_config_handler,
            .user_ctx  = NULL
        };
        httpd_register_uri_handler(server, &wifi_config);

        return server;
    }

    ESP_LOGI(TAG, "Error starting server!");
    return NULL;
}
```

## 🌍 TCP/UDP网络编程

### TCP客户端

```c
#include "lwip/err.h"
#include "lwip/sockets.h"
#include "lwip/sys.h"
#include "lwip/netdb.h"
#include "lwip/dns.h"

#define HOST_IP_ADDR "192.168.1.100"
#define PORT 3333

static void tcp_client_task(void *pvParameters)
{
    char rx_buffer[128];
    char host_ip[] = HOST_IP_ADDR;
    int addr_family = 0;
    int ip_protocol = 0;

    while (1) {
        struct sockaddr_in dest_addr;
        dest_addr.sin_addr.s_addr = inet_addr(host_ip);
        dest_addr.sin_family = AF_INET;
        dest_addr.sin_port = htons(PORT);
        addr_family = AF_INET;
        ip_protocol = IPPROTO_IP;

        int sock =  socket(addr_family, SOCK_STREAM, ip_protocol);
        if (sock < 0) {
            ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
            break;
        }
        ESP_LOGI(TAG, "Socket created, connecting to %s:%d", host_ip, PORT);

        int err = connect(sock, (struct sockaddr *)&dest_addr, sizeof(dest_addr));
        if (err != 0) {
            ESP_LOGE(TAG, "Socket unable to connect: errno %d", errno);
            close(sock);
            vTaskDelay(4000 / portTICK_PERIOD_MS);
            continue;
        }
        ESP_LOGI(TAG, "Successfully connected");

        while (1) {
            // 发送数据
            const char *payload = "Hello from ESP32!";
            int err = send(sock, payload, strlen(payload), 0);
            if (err < 0) {
                ESP_LOGE(TAG, "Error occurred during sending: errno %d", errno);
                break;
            }

            // 接收数据
            int len = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
            if (len < 0) {
                ESP_LOGE(TAG, "recv failed: errno %d", errno);
                break;
            } else if (len == 0) {
                ESP_LOGW(TAG, "Connection closed");
                break;
            } else {
                rx_buffer[len] = 0;
                ESP_LOGI(TAG, "Received %d bytes: %s", len, rx_buffer);
            }

            vTaskDelay(2000 / portTICK_PERIOD_MS);
        }

        if (sock != -1) {
            ESP_LOGE(TAG, "Shutting down socket and restarting...");
            shutdown(sock, 0);
            close(sock);
        }
    }
    vTaskDelete(NULL);
}
```

### TCP服务器

```c
static void tcp_server_task(void *pvParameters)
{
    char rx_buffer[128];
    char addr_str[128];
    int addr_family = AF_INET;
    int ip_protocol = IPPROTO_IP;
    int keepAlive = 1;
    int keepIdle = 7200;
    int keepInterval = 75;
    int keepCount = 9;

    struct sockaddr_storage dest_addr;
    struct sockaddr_in *dest_addr_ip4 = (struct sockaddr_in *)&dest_addr;
    dest_addr_ip4->sin_addr.s_addr = htonl(INADDR_ANY);
    dest_addr_ip4->sin_family = AF_INET;
    dest_addr_ip4->sin_port = htons(PORT);

    int listen_sock = socket(addr_family, SOCK_STREAM, ip_protocol);
    if (listen_sock < 0) {
        ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
        vTaskDelete(NULL);
        return;
    }

    int opt = 1;
    setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    ESP_LOGI(TAG, "Socket created");

    int err = bind(listen_sock, (struct sockaddr *)&dest_addr, sizeof(dest_addr));
    if (err != 0) {
        ESP_LOGE(TAG, "Socket unable to bind: errno %d", errno);
        ESP_LOGE(TAG, "IPPROTO: %d", addr_family);
        goto CLEAN_UP;
    }
    ESP_LOGI(TAG, "Socket bound, port %d", PORT);

    err = listen(listen_sock, 1);
    if (err != 0) {
        ESP_LOGE(TAG, "Error occurred during listen: errno %d", errno);
        goto CLEAN_UP;
    }

    while (1) {
        ESP_LOGI(TAG, "Socket listening");

        struct sockaddr_storage source_addr;
        socklen_t addr_len = sizeof(source_addr);
        int sock = accept(listen_sock, (struct sockaddr *)&source_addr, &addr_len);
        if (sock < 0) {
            ESP_LOGE(TAG, "Unable to accept connection: errno %d", errno);
            break;
        }

        // 设置TCP keep-alive参数
        setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, &keepAlive, sizeof(int));
        setsockopt(sock, IPPROTO_TCP, TCP_KEEPIDLE, &keepIdle, sizeof(int));
        setsockopt(sock, IPPROTO_TCP, TCP_KEEPINTVL, &keepInterval, sizeof(int));
        setsockopt(sock, IPPROTO_TCP, TCP_KEEPCNT, &keepCount, sizeof(int));

        // 转换地址为字符串
        if (source_addr.ss_family == PF_INET) {
            inet_ntoa_r(((struct sockaddr_in *)&source_addr)->sin_addr, addr_str, sizeof(addr_str) - 1);
        }
        ESP_LOGI(TAG, "Socket accepted ip address: %s", addr_str);

        while (1) {
            int len = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
            if (len < 0) {
                ESP_LOGE(TAG, "Error occurred during receiving: errno %d", errno);
                break;
            } else if (len == 0) {
                ESP_LOGW(TAG, "Connection closed");
                break;
            } else {
                rx_buffer[len] = 0;
                ESP_LOGI(TAG, "Received %d bytes from %s: %s", len, addr_str, rx_buffer);

                // 回显数据
                int to_write = len;
                while (to_write > 0) {
                    int written = send(sock, rx_buffer + (len - to_write), to_write, 0);
                    if (written < 0) {
                        ESP_LOGE(TAG, "Error occurred during sending: errno %d", errno);
                        goto client_cleanup;
                    }
                    to_write -= written;
                }
            }
        }

client_cleanup:
        shutdown(sock, 0);
        close(sock);
    }

CLEAN_UP:
    close(listen_sock);
    vTaskDelete(NULL);
}
```

### UDP通信

```c
static void udp_server_task(void *pvParameters)
{
    char rx_buffer[128];
    char addr_str[128];
    int addr_family = AF_INET;
    int ip_protocol = IPPROTO_IP;

    struct sockaddr_in dest_addr;
    dest_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    dest_addr.sin_family = AF_INET;
    dest_addr.sin_port = htons(PORT);

    int sock = socket(addr_family, SOCK_DGRAM, ip_protocol);
    if (sock < 0) {
        ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
        return;
    }
    ESP_LOGI(TAG, "Socket created");

    int err = bind(sock, (struct sockaddr *)&dest_addr, sizeof(dest_addr));
    if (err < 0) {
        ESP_LOGE(TAG, "Socket unable to bind: errno %d", errno);
        goto cleanup;
    }
    ESP_LOGI(TAG, "Socket bound, port %d", PORT);

    while (1) {
        struct sockaddr_storage source_addr;
        socklen_t socklen = sizeof(source_addr);
        
        int len = recvfrom(sock, rx_buffer, sizeof(rx_buffer) - 1, 0, 
                          (struct sockaddr *)&source_addr, &socklen);

        if (len < 0) {
            ESP_LOGE(TAG, "recvfrom failed: errno %d", errno);
            break;
        } else {
            rx_buffer[len] = 0;

            // 获取发送方地址
            if (source_addr.ss_family == PF_INET) {
                inet_ntoa_r(((struct sockaddr_in *)&source_addr)->sin_addr, 
                           addr_str, sizeof(addr_str) - 1);
            }

            ESP_LOGI(TAG, "Received %d bytes from %s: %s", len, addr_str, rx_buffer);

            // 回复数据
            const char* reply = "Hello from ESP32 UDP server!";
            int err = sendto(sock, reply, strlen(reply), 0, 
                           (struct sockaddr *)&source_addr, sizeof(source_addr));
            if (err < 0) {
                ESP_LOGE(TAG, "Error occurred during sending: errno %d", errno);
                break;
            }
        }
    }

cleanup:
    if (sock != -1) {
        ESP_LOGE(TAG, "Shutting down socket");
        shutdown(sock, 0);
        close(sock);
    }
    vTaskDelete(NULL);
}
```

## 🔐 网络安全和加密

### TLS/SSL连接

```c
#include "esp_tls.h"

static void https_client_task(void *pvParameters)
{
    esp_tls_cfg_t cfg = {
        .crt_bundle_attach = esp_crt_bundle_attach,
        .timeout_ms = 5000,
    };

    esp_tls_t *tls = esp_tls_conn_http_new("https://httpbin.org/get", &cfg);
    
    if (tls != NULL) {
        ESP_LOGI(TAG, "Connection established...");
    } else {
        ESP_LOGE(TAG, "Connection failed...");
        goto exit;
    }

    const char *GET_REQUEST = "GET /get HTTP/1.1\r\n"
                             "Host: httpbin.org\r\n"
                             "User-Agent: ESP32\r\n"
                             "\r\n";

    size_t written_bytes = 0;
    do {
        int ret = esp_tls_conn_write(tls, GET_REQUEST + written_bytes, 
                                    strlen(GET_REQUEST) - written_bytes);
        if (ret >= 0) {
            ESP_LOGI(TAG, "%d bytes written", ret);
            written_bytes += ret;
        } else if (ret != ESP_TLS_ERR_SSL_WANT_READ && ret != ESP_TLS_ERR_SSL_WANT_WRITE) {
            ESP_LOGE(TAG, "esp_tls_conn_write  returned 0x%x", ret);
            goto exit;
        }
    } while (written_bytes < strlen(GET_REQUEST));

    ESP_LOGI(TAG, "Reading HTTP response...");
    char buf[512];
    int ret;
    do {
        int len = sizeof(buf) - 1;
        bzero(buf, sizeof(buf));
        ret = esp_tls_conn_read(tls, (char *)buf, len);

        if (ret == ESP_TLS_ERR_SSL_WANT_WRITE || ret == ESP_TLS_ERR_SSL_WANT_READ) {
            continue;
        }

        if (ret < 0) {
            ESP_LOGE(TAG, "esp_tls_conn_read  returned -0x%x", -ret);
            break;
        }

        if (ret == 0) {
            ESP_LOGI(TAG, "connection closed");
            break;
        }

        len = ret;
        ESP_LOGD(TAG, "%d bytes read", len);
        for (int i = 0; i < len; i++) {
            putchar(buf[i]);
        }
    } while (1);

exit:
    esp_tls_conn_delete(tls);
    vTaskDelete(NULL);
}
```

---

> **WiFi与网络编程总结**：ESP32的WiFi功能非常强大，支持多种工作模式和网络协议。从基础的STA/AP配置到高级的TCP/UDP编程，再到安全的TLS连接，为IoT应用提供了完整的网络解决方案。合理使用这些功能可以构建稳定可靠的物联网设备。
