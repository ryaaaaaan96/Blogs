# ESP32蓝牙开发指南

## 📋 概述

ESP32内置了经典蓝牙（Classic Bluetooth）和低功耗蓝牙（Bluetooth Low Energy, BLE）功能。BLE特别适合于电池供电的IoT设备，具有低功耗、快速连接等优势。本文将全面介绍ESP32的蓝牙开发。

## 📡 蓝牙基础知识

### 蓝牙技术对比

| 特性 | 经典蓝牙 | 低功耗蓝牙(BLE) |
|------|----------|-----------------|
| 功耗 | 高 | 极低 |
| 连接速度 | 慢(3-5秒) | 快(几毫秒) |
| 数据传输速率 | 高(1-3 Mbps) | 低(1 Mbps) |
| 连接距离 | 10-100米 | 10-50米 |
| 应用场景 | 音频流、文件传输 | 传感器、可穿戴设备 |
| 电池寿命 | 天-周 | 月-年 |

### BLE协议栈架构

```
应用层 (Application Layer)
    ↕
通用访问配置文件 (GAP - Generic Access Profile)
通用属性配置文件 (GATT - Generic Attribute Profile)
    ↕
属性协议 (ATT - Attribute Protocol)
安全管理协议 (SMP - Security Manager Protocol)
    ↕
逻辑链路控制和适配协议 (L2CAP)
    ↕
主机控制接口 (HCI - Host Controller Interface)
    ↕
链路层 (Link Layer)
    ↕
物理层 (Physical Layer)
```

## 🔧 BLE基础配置

### BLE初始化

```c
#include "esp_bt.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_defs.h"
#include "esp_bt_main.h"
#include "esp_gatt_common_api.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "nvs_flash.h"

static const char *TAG = "BLE_GATTS";

// GATTS配置参数
#define GATTS_TABLE_TAG  "GATTS_DEMO"
#define PROFILE_NUM      1
#define PROFILE_APP_IDX  0
#define ESP_APP_ID       0x55
#define SAMPLE_DEVICE_NAME   "ESP_GATTS_DEMO"
#define SVC_INST_ID      0

// 服务和特征值UUID
#define GATTS_SERVICE_UUID   0x00FF
#define GATTS_CHAR_UUID      0xFF01
#define GATTS_DESCR_UUID     0x3333
#define GATTS_NUM_HANDLE     4

#define GATTS_CHAR_VAL_LEN_MAX 500
#define PREPARE_BUF_MAX_SIZE   1024

static uint8_t adv_config_done       = 0;
#define ADV_CONFIG_FLAG      (1 << 0)
#define SCAN_RSP_CONFIG_FLAG (1 << 1)

// 广播数据
static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp        = false,
    .include_name        = true,
    .include_txpower     = true,
    .min_interval        = 0x0006, // slave connection min interval
    .max_interval        = 0x0010, // slave connection max interval
    .appearance          = 0x00,
    .manufacturer_len    = 0,
    .p_manufacturer_data = NULL,
    .service_data_len    = 0,
    .p_service_data      = NULL,
    .service_uuid_len    = sizeof(uint16_t),
    .p_service_uuid      = (uint8_t[]){GATTS_SERVICE_UUID & 0xFF, (GATTS_SERVICE_UUID >> 8) & 0xFF},
    .flag = (ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT),
};

// 扫描响应数据
static esp_ble_adv_data_t scan_rsp_data = {
    .set_scan_rsp        = true,
    .include_name        = true,
    .include_txpower     = true,
    .appearance          = 0x00,
    .manufacturer_len    = 0,
    .p_manufacturer_data = NULL,
    .service_data_len    = 0,
    .p_service_data      = NULL,
    .service_uuid_len    = 0,
    .p_service_uuid      = NULL,
    .flag = (ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT),
};

// 广播参数
static esp_ble_adv_params_t adv_params = {
    .adv_int_min         = 0x20,
    .adv_int_max         = 0x40,
    .adv_type            = ADV_TYPE_IND,
    .own_addr_type       = BLE_ADDR_TYPE_PUBLIC,
    .channel_map         = ADV_CHNL_ALL,
    .adv_filter_policy   = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

// GATT接口结构
struct gatts_profile_inst {
    esp_gatts_cb_t gatts_cb;
    uint16_t gatts_if;
    uint16_t app_id;
    uint16_t conn_id;
    uint16_t service_handle;
    esp_gatt_srvc_id_t service_id;
    uint16_t char_handle;
    esp_bt_uuid_t char_uuid;
    esp_gatt_perm_t perm;
    esp_gatt_char_prop_t property;
    uint16_t descr_handle;
    esp_bt_uuid_t descr_uuid;
};

static void gatts_profile_event_handler(esp_gatts_cb_event_t event,
					esp_gatt_if_t gatts_if, esp_ble_gatts_cb_param_t *param);

// 配置文件表
static struct gatts_profile_inst heart_rate_profile_tab[PROFILE_NUM] = {
    [PROFILE_APP_IDX] = {
        .gatts_cb = gatts_profile_event_handler,
        .gatts_if = ESP_GATT_IF_NONE,
    },
};

// BLE初始化函数
esp_err_t ble_init(void)
{
    esp_err_t ret;

    // 初始化NVS
    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 释放经典蓝牙内存
    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    // 初始化蓝牙控制器
    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ret = esp_bt_controller_init(&bt_cfg);
    if (ret) {
        ESP_LOGE(TAG, "%s enable controller failed: %s", __func__, esp_err_to_name(ret));
        return ret;
    }

    // 启用蓝牙控制器
    ret = esp_bt_controller_enable(ESP_BT_MODE_BLE);
    if (ret) {
        ESP_LOGE(TAG, "%s enable controller failed: %s", __func__, esp_err_to_name(ret));
        return ret;
    }

    // 初始化蓝牙主机栈
    ret = esp_bluedroid_init();
    if (ret) {
        ESP_LOGE(TAG, "%s init bluetooth failed: %s", __func__, esp_err_to_name(ret));
        return ret;
    }

    // 启用蓝牙主机栈
    ret = esp_bluedroid_enable();
    if (ret) {
        ESP_LOGE(TAG, "%s enable bluetooth failed: %s", __func__, esp_err_to_name(ret));
        return ret;
    }

    // 注册GATTS回调函数
    ret = esp_ble_gatts_register_callback(gatts_event_handler);
    if (ret){
        ESP_LOGE(TAG, "gatts register error, error code = %x", ret);
        return ret;
    }

    // 注册GAP回调函数
    ret = esp_ble_gap_register_callback(gap_event_handler);
    if (ret){
        ESP_LOGE(TAG, "gap register error, error code = %x", ret);
        return ret;
    }

    // 注册应用程序配置文件
    ret = esp_ble_gatts_app_register(ESP_APP_ID);
    if (ret){
        ESP_LOGE(TAG, "gatts app register error, error code = %x", ret);
        return ret;
    }

    // 设置MTU大小
    esp_err_t local_mtu_ret = esp_ble_gatt_set_local_mtu(517);
    if (local_mtu_ret){
        ESP_LOGE(TAG, "set local  MTU failed, error code = %x", local_mtu_ret);
    }

    return ESP_OK;
}
```

## 📶 GAP（通用访问配置文件）

### 广播和扫描

```c
// GAP事件处理器
static void gap_event_handler(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_SET_COMPLETE_EVT:
        adv_config_done &= (~ADV_CONFIG_FLAG);
        if (adv_config_done == 0){
            esp_ble_gap_start_advertising(&adv_params);
        }
        break;
    case ESP_GAP_BLE_SCAN_RSP_DATA_SET_COMPLETE_EVT:
        adv_config_done &= (~SCAN_RSP_CONFIG_FLAG);
        if (adv_config_done == 0){
            esp_ble_gap_start_advertising(&adv_params);
        }
        break;
    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status != ESP_BT_STATUS_SUCCESS) {
            ESP_LOGE(TAG, "advertising start failed");
        } else {
            ESP_LOGI(TAG, "advertising start successfully");
        }
        break;
    case ESP_GAP_BLE_ADV_STOP_COMPLETE_EVT:
        if (param->adv_stop_cmpl.status != ESP_BT_STATUS_SUCCESS) {
            ESP_LOGE(TAG, "Advertising stop failed");
        } else {
            ESP_LOGI(TAG, "Stop adv successfully");
        }
        break;
    case ESP_GAP_BLE_UPDATE_CONN_PARAMS_EVT:
        ESP_LOGI(TAG, "update connection params status = %d, min_int = %d, max_int = %d,conn_int = %d,latency = %d, timeout = %d",
                  param->update_conn_params.status,
                  param->update_conn_params.min_int,
                  param->update_conn_params.max_int,
                  param->update_conn_params.conn_int,
                  param->update_conn_params.latency,
                  param->update_conn_params.timeout);
        break;
    default:
        break;
    }
}

// 开始BLE扫描
void start_ble_scan(void)
{
    esp_ble_scan_params_t ble_scan_params = {
        .scan_type              = BLE_SCAN_TYPE_ACTIVE,
        .own_addr_type          = BLE_ADDR_TYPE_PUBLIC,
        .scan_filter_policy     = BLE_SCAN_FILTER_ALLOW_ALL,
        .scan_interval          = 0x50,
        .scan_window            = 0x30,
        .scan_duplicate         = BLE_SCAN_DUPLICATE_DISABLE
    };

    esp_ble_gap_set_scan_params(&ble_scan_params);
    esp_ble_gap_start_scanning(30); // 扫描30秒
}

// 扫描结果处理
static void scan_result_handler(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_SCAN_RESULT_EVT: {
        esp_ble_gap_cb_param_t *scan_result = (esp_ble_gap_cb_param_t *)param;
        switch (scan_result->scan_rst.search_evt) {
        case ESP_GAP_SEARCH_INQ_RES_EVT:
            esp_log_buffer_hex(TAG, scan_result->scan_rst.bda, 6);
            ESP_LOGI(TAG, "searched Adv Data Len %d, Scan Response Len %d", 
                     scan_result->scan_rst.adv_data_len, scan_result->scan_rst.scan_rsp_len);
            
            // 解析广播数据
            uint8_t *adv_name = NULL;
            uint8_t adv_name_len = 0;
            adv_name = esp_ble_resolve_adv_data(scan_result->scan_rst.ble_adv,
                                               ESP_BLE_AD_TYPE_NAME_CMPL, &adv_name_len);
            ESP_LOGI(TAG, "searched Device Name Len %d", adv_name_len);
            esp_log_buffer_char(TAG, adv_name, adv_name_len);
            ESP_LOGI(TAG, "");
            
            // RSSI强度
            ESP_LOGI(TAG, "RSSI: %d dBm", scan_result->scan_rst.rssi);
            break;
        case ESP_GAP_SEARCH_INQ_CMPL_EVT:
            ESP_LOGI(TAG, "BLE scan complete");
            break;
        default:
            break;
        }
        break;
    }
    default:
        break;
    }
}
```

### 连接参数管理

```c
// 连接参数配置
typedef struct {
    uint16_t min_conn_interval;    // 最小连接间隔
    uint16_t max_conn_interval;    // 最大连接间隔
    uint16_t slave_latency;        // 从设备延迟
    uint16_t supervision_timeout;  // 监督超时
} ble_conn_params_t;

static ble_conn_params_t conn_params = {
    .min_conn_interval = 0x10,     // 20ms
    .max_conn_interval = 0x20,     // 40ms
    .slave_latency = 0,
    .supervision_timeout = 0x64,   // 1000ms
};

// 更新连接参数
esp_err_t update_connection_params(uint16_t conn_id)
{
    esp_ble_conn_update_params_t conn_params_update = {
        .min_int = conn_params.min_conn_interval,
        .max_int = conn_params.max_conn_interval,
        .latency = conn_params.slave_latency,
        .timeout = conn_params.supervision_timeout,
    };
    
    // 获取连接的远程设备地址
    esp_bd_addr_t remote_bda;
    esp_ble_gap_get_remote_device_name(remote_bda); // 需要从连接事件中获取
    
    return esp_ble_gap_update_conn_params(&conn_params_update);
}

// 设备配对和绑定
void init_ble_security(void)
{
    // 设置IO能力
    esp_ble_io_cap_t iocap = ESP_IO_CAP_NONE;
    esp_ble_gap_set_security_param(ESP_BLE_SM_IOCAP_MODE, &iocap, sizeof(uint8_t));

    // 设置认证要求
    uint8_t auth_req = ESP_LE_AUTH_REQ_SC_MITM_BOND;
    esp_ble_gap_set_security_param(ESP_BLE_SM_AUTHEN_REQ_MODE, &auth_req, sizeof(uint8_t));

    // 设置加密密钥大小
    uint8_t key_size = 16;
    esp_ble_gap_set_security_param(ESP_BLE_SM_MAX_KEY_SIZE, &key_size, sizeof(uint8_t));

    // 设置初始密钥分发
    uint8_t init_key = ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK;
    esp_ble_gap_set_security_param(ESP_BLE_SM_SET_INIT_KEY, &init_key, sizeof(uint8_t));

    // 设置响应密钥分发
    uint8_t rsp_key = ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK;
    esp_ble_gap_set_security_param(ESP_BLE_SM_SET_RSP_KEY, &rsp_key, sizeof(uint8_t));
}
```

## 🏗️ GATT服务器开发

### 服务和特征值定义

```c
// 服务表枚举
enum {
    HEART_RATE_SVC_IDX,                // 心率服务
    HEART_RATE_CHAR_IDX,               // 心率测量特征值
    HEART_RATE_CHAR_VAL_IDX,           // 心率测量值
    HEART_RATE_CHAR_CFG_IDX,           // 客户端特征配置描述符

    BATTERY_SVC_IDX,                   // 电池服务
    BATTERY_CHAR_IDX,                  // 电池电量特征值
    BATTERY_CHAR_VAL_IDX,              // 电池电量值

    HRS_IDX_NB,                        // 总数量
};

// 服务表定义
static const uint16_t heart_rate_svc = ESP_GATT_UUID_HEART_RATE_SVC;
static const uint16_t heart_rate_meas_char = ESP_GATT_UUID_HEART_RATE_MEAS;
static const uint16_t heart_rate_ccc = ESP_GATT_UUID_CHAR_CLIENT_CONFIG;

static const uint16_t battery_svc = ESP_GATT_UUID_BATTERY_SERVICE_SVC;
static const uint16_t battery_level_char = ESP_GATT_UUID_BATTERY_LEVEL;

static const uint16_t primary_service_uuid = ESP_GATT_UUID_PRI_SERVICE;
static const uint16_t character_declaration_uuid = ESP_GATT_UUID_CHAR_DECLARE;
static const uint16_t character_client_config_uuid = ESP_GATT_UUID_CHAR_CLIENT_CONFIG;

static const uint8_t char_prop_notify = ESP_GATT_CHAR_PROP_BIT_NOTIFY;
static const uint8_t char_prop_read = ESP_GATT_CHAR_PROP_BIT_READ;
static const uint8_t char_prop_read_write = ESP_GATT_CHAR_PROP_BIT_WRITE | ESP_GATT_CHAR_PROP_BIT_READ;

// 心率测量值
static uint8_t heart_measurement_ccc[2] = {0x00, 0x00};
static uint8_t heart_rate_value[4] = {0x00, 0x5A, 0x00, 0x00}; // 心率90 BPM

// 电池电量值
static uint8_t battery_level = 85; // 85%

// 属性数据库
static const esp_gatts_attr_db_t heart_rate_gatt_db[HRS_IDX_NB] = {
    // 心率服务声明
    [HEART_RATE_SVC_IDX] = {
        {ESP_GATT_AUTO_RSP}, 
        {ESP_UUID_LEN_16, (uint8_t *)&primary_service_uuid, ESP_GATT_PERM_READ,
         sizeof(uint16_t), sizeof(heart_rate_svc), (uint8_t *)&heart_rate_svc}
    },

    // 心率测量特征值声明
    [HEART_RATE_CHAR_IDX] = {
        {ESP_GATT_AUTO_RSP}, 
        {ESP_UUID_LEN_16, (uint8_t *)&character_declaration_uuid, ESP_GATT_PERM_READ,
         sizeof(char_prop_notify), sizeof(char_prop_notify), (uint8_t *)&char_prop_notify}
    },

    // 心率测量特征值
    [HEART_RATE_CHAR_VAL_IDX] = {
        {ESP_GATT_AUTO_RSP}, 
        {ESP_UUID_LEN_16, (uint8_t *)&heart_rate_meas_char, ESP_GATT_PERM_READ,
         GATTS_CHAR_VAL_LEN_MAX, sizeof(heart_rate_value), heart_rate_value}
    },

    // 客户端特征配置描述符
    [HEART_RATE_CHAR_CFG_IDX] = {
        {ESP_GATT_AUTO_RSP}, 
        {ESP_UUID_LEN_16, (uint8_t *)&character_client_config_uuid, 
         ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
         sizeof(uint16_t), sizeof(heart_measurement_ccc), (uint8_t *)heart_measurement_ccc}
    },

    // 电池服务声明
    [BATTERY_SVC_IDX] = {
        {ESP_GATT_AUTO_RSP}, 
        {ESP_UUID_LEN_16, (uint8_t *)&primary_service_uuid, ESP_GATT_PERM_READ,
         sizeof(uint16_t), sizeof(battery_svc), (uint8_t *)&battery_svc}
    },

    // 电池电量特征值声明
    [BATTERY_CHAR_IDX] = {
        {ESP_GATT_AUTO_RSP}, 
        {ESP_UUID_LEN_16, (uint8_t *)&character_declaration_uuid, ESP_GATT_PERM_READ,
         sizeof(char_prop_read), sizeof(char_prop_read), (uint8_t *)&char_prop_read}
    },

    // 电池电量特征值
    [BATTERY_CHAR_VAL_IDX] = {
        {ESP_GATT_AUTO_RSP}, 
        {ESP_UUID_LEN_16, (uint8_t *)&battery_level_char, ESP_GATT_PERM_READ,
         sizeof(uint8_t), sizeof(battery_level), &battery_level}
    },
};
```

### GATTS事件处理

```c
// GATTS配置文件事件处理器
static void gatts_profile_event_handler(esp_gatts_cb_event_t event, 
                                       esp_gatt_if_t gatts_if, 
                                       esp_ble_gatts_cb_param_t *param)
{
    switch (event) {
    case ESP_GATTS_REG_EVT: {
        esp_err_t set_dev_name_ret = esp_ble_gap_set_device_name(SAMPLE_DEVICE_NAME);
        if (set_dev_name_ret){
            ESP_LOGE(TAG, "set device name failed, error code = %x", set_dev_name_ret);
        }

        // 配置广播数据
        esp_err_t ret = esp_ble_gap_config_adv_data(&adv_data);
        if (ret){
            ESP_LOGE(TAG, "config adv data failed, error code = %x", ret);
        }
        adv_config_done |= ADV_CONFIG_FLAG;

        // 配置扫描响应数据
        ret = esp_ble_gap_config_adv_data(&scan_rsp_data);
        if (ret){
            ESP_LOGE(TAG, "config scan response data failed, error code = %x", ret);
        }
        adv_config_done |= SCAN_RSP_CONFIG_FLAG;

        // 创建属性表
        esp_err_t create_attr_ret = esp_ble_gatts_create_attr_tab(heart_rate_gatt_db, 
                                                                 gatts_if, 
                                                                 HRS_IDX_NB, 
                                                                 SVC_INST_ID);
        if (create_attr_ret){
            ESP_LOGE(TAG, "create attr table failed, error code = %x", create_attr_ret);
        }
    }
    break;

    case ESP_GATTS_READ_EVT:
        ESP_LOGI(TAG, "ESP_GATTS_READ_EVT");
        break;

    case ESP_GATTS_WRITE_EVT:
        if (!param->write.is_prep){
            ESP_LOGI(TAG, "GATT_WRITE_EVT, handle = %d, value len = %d, value :", 
                     param->write.handle, param->write.len);
            esp_log_buffer_hex(TAG, param->write.value, param->write.len);
            
            // 处理客户端特征配置描述符写入
            if (heart_rate_handle_table[HEART_RATE_CHAR_CFG_IDX] == param->write.handle && 
                param->write.len == 2){
                uint16_t descr_value = param->write.value[1]<<8 | param->write.value[0];
                if (descr_value == 0x0001){
                    ESP_LOGI(TAG, "notify enable");
                    // 开始发送通知
                    start_heart_rate_notification(gatts_if, param->write.conn_id);
                } else if (descr_value == 0x0002){
                    ESP_LOGI(TAG, "indicate enable");
                } else if (descr_value == 0x0000){
                    ESP_LOGI(TAG, "notify/indicate disable");
                    stop_heart_rate_notification();
                } else {
                    ESP_LOGE(TAG, "unknown descr value");
                    esp_log_buffer_hex(TAG, param->write.value, param->write.len);
                }
            }

            // 发送写入响应
            if (param->write.need_rsp){
                esp_ble_gatts_send_response(gatts_if, param->write.conn_id, 
                                          param->write.trans_id, ESP_GATT_OK, NULL);
            }
        } else {
            // 处理长写入的准备阶段
            example_prepare_write_event_env(gatts_if, &prepare_write_env, param);
        }
        break;

    case ESP_GATTS_EXEC_WRITE_EVT: 
        ESP_LOGI(TAG, "ESP_GATTS_EXEC_WRITE_EVT");
        example_exec_write_event_env(&prepare_write_env, param);
        break;

    case ESP_GATTS_MTU_EVT:
        ESP_LOGI(TAG, "ESP_GATTS_MTU_EVT, MTU %d", param->mtu.mtu);
        break;

    case ESP_GATTS_CONF_EVT:
        ESP_LOGI(TAG, "ESP_GATTS_CONF_EVT, status = %d, attr_handle %d", 
                 param->conf.status, param->conf.handle);
        break;

    case ESP_GATTS_START_EVT:
        ESP_LOGI(TAG, "SERVICE_START_EVT, status %d, service_handle %d", 
                 param->start.status, param->start.service_handle);
        break;

    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "ESP_GATTS_CONNECT_EVT, conn_id = %d", param->connect.conn_id);
        esp_log_buffer_hex(TAG, param->connect.remote_bda, 6);
        
        heart_rate_profile_tab[PROFILE_APP_IDX].conn_id = param->connect.conn_id;
        
        // 更新连接参数
        esp_ble_conn_update_params_t conn_params = {0};
        memcpy(conn_params.bda, param->connect.remote_bda, sizeof(esp_bd_addr_t));
        conn_params.latency = 0;
        conn_params.max_int = 0x20;    // max_int = 0x20*1.25ms = 40ms
        conn_params.min_int = 0x10;    // min_int = 0x10*1.25ms = 20ms
        conn_params.timeout = 400;     // timeout = 400*10ms = 4000ms
        esp_ble_gap_update_conn_params(&conn_params);
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "ESP_GATTS_DISCONNECT_EVT, reason = 0x%x", param->disconnect.reason);
        stop_heart_rate_notification();
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GATTS_CREAT_ATTR_TAB_EVT:{
        if (param->add_attr_tab.status != ESP_GATT_OK){
            ESP_LOGE(TAG, "create attribute table failed, error code=0x%x", param->add_attr_tab.status);
        }
        else if (param->add_attr_tab.num_handle != HRS_IDX_NB){
            ESP_LOGE(TAG, "create attribute table abnormally, num_handle (%d) doesn't equal to HRS_IDX_NB(%d)", 
                     param->add_attr_tab.num_handle, HRS_IDX_NB);
        }
        else {
            ESP_LOGI(TAG, "create attribute table successfully, the number handle = %d",param->add_attr_tab.num_handle);
            memcpy(heart_rate_handle_table, param->add_attr_tab.handles, sizeof(heart_rate_handle_table));
            esp_ble_gatts_start_service(heart_rate_handle_table[HEART_RATE_SVC_IDX]);
        }
        break;
    }

    default:
        break;
    }
}
```

### 数据通知和指示

```c
static bool notify_enabled = false;
static TimerHandle_t heart_rate_timer = NULL;

// 心率数据通知定时器回调
static void heart_rate_timer_callback(TimerHandle_t xTimer)
{
    if (notify_enabled && heart_rate_profile_tab[PROFILE_APP_IDX].gatts_if != ESP_GATT_IF_NONE) {
        // 模拟心率数据变化
        static uint8_t heart_rate = 60;
        heart_rate += (rand() % 20) - 10; // ±10 BPM变化
        if (heart_rate < 50) heart_rate = 50;
        if (heart_rate > 180) heart_rate = 180;
        
        // 构造心率测量数据包
        uint8_t heart_rate_data[4];
        heart_rate_data[0] = 0x00; // Flags: 心率值格式为uint8
        heart_rate_data[1] = heart_rate;
        heart_rate_data[2] = 0x00; // RR间隔不存在
        heart_rate_data[3] = 0x00;
        
        // 发送通知
        esp_ble_gatts_send_indicate(heart_rate_profile_tab[PROFILE_APP_IDX].gatts_if,
                                   heart_rate_profile_tab[PROFILE_APP_IDX].conn_id,
                                   heart_rate_handle_table[HEART_RATE_CHAR_VAL_IDX],
                                   sizeof(heart_rate_data),
                                   heart_rate_data,
                                   false); // false = notification, true = indication
        
        ESP_LOGI(TAG, "Heart rate notification sent: %d BPM", heart_rate);
    }
}

// 开始心率通知
void start_heart_rate_notification(esp_gatt_if_t gatts_if, uint16_t conn_id)
{
    notify_enabled = true;
    
    if (heart_rate_timer == NULL) {
        heart_rate_timer = xTimerCreate("HeartRateTimer",
                                      pdMS_TO_TICKS(1000), // 1秒周期
                                      pdTRUE,              // 自动重载
                                      NULL,
                                      heart_rate_timer_callback);
    }
    
    if (heart_rate_timer != NULL) {
        xTimerStart(heart_rate_timer, 0);
        ESP_LOGI(TAG, "Heart rate notification started");
    }
}

// 停止心率通知
void stop_heart_rate_notification(void)
{
    notify_enabled = false;
    
    if (heart_rate_timer != NULL) {
        xTimerStop(heart_rate_timer, 0);
        ESP_LOGI(TAG, "Heart rate notification stopped");
    }
}
```

## 📱 BLE客户端开发

### 客户端连接和服务发现

```c
#include "esp_gattc_api.h"

#define REMOTE_SERVICE_UUID        0x00FF
#define REMOTE_NOTIFY_CHAR_UUID    0xFF01
#define PROFILE_NUM                1
#define PROFILE_A_APP_ID           0
#define INVALID_HANDLE             0

static bool connect = false;
static bool get_server = false;
static esp_gattc_char_elem_t  *char_elem_result   = NULL;
static esp_gattc_descr_elem_t *descr_elem_result  = NULL;

// GATTC配置文件结构
struct gattc_profile_inst {
    esp_gattc_cb_t gattc_cb;
    uint16_t gattc_if;
    uint16_t app_id;
    uint16_t conn_id;
    uint16_t service_start_handle;
    uint16_t service_end_handle;
    uint16_t char_handle;
    esp_bd_addr_t remote_bda;
};

static struct gattc_profile_inst gl_profile_tab[PROFILE_NUM] = {
    [PROFILE_A_APP_ID] = {
        .gattc_cb = gattc_profile_event_handler,
        .gattc_if = ESP_GATT_IF_NONE,
    },
};

// GATTC事件处理器
static void gattc_profile_event_handler(esp_gattc_cb_event_t event, 
                                       esp_gatt_if_t gattc_if, 
                                       esp_ble_gattc_cb_param_t *param)
{
    esp_ble_gattc_cb_param_t *p_data = (esp_ble_gattc_cb_param_t *)param;

    switch (event) {
    case ESP_GATTC_REG_EVT:
        ESP_LOGI(TAG, "REG_EVT");
        esp_err_t scan_ret = esp_ble_gap_set_scan_params(&ble_scan_params);
        if (scan_ret){
            ESP_LOGE(TAG, "set scan params error, error code = %x", scan_ret);
        }
        break;

    case ESP_GATTC_CONNECT_EVT:{
        ESP_LOGI(TAG, "ESP_GATTC_CONNECT_EVT conn_id %d, if %d", p_data->connect.conn_id, gattc_if);
        gl_profile_tab[PROFILE_A_APP_ID].conn_id = p_data->connect.conn_id;
        memcpy(gl_profile_tab[PROFILE_A_APP_ID].remote_bda, p_data->connect.remote_bda, sizeof(esp_bd_addr_t));
        ESP_LOGI(TAG, "REMOTE BDA:");
        esp_log_buffer_hex(TAG, gl_profile_tab[PROFILE_A_APP_ID].remote_bda, sizeof(esp_bd_addr_t));
        
        esp_err_t mtu_ret = esp_ble_gattc_send_mtu_req(gattc_if, p_data->connect.conn_id);
        if (mtu_ret){
            ESP_LOGE(TAG, "config MTU error, error code = %x", mtu_ret);
        }
        break;
    }

    case ESP_GATTC_OPEN_EVT:
        if (param->open.status != ESP_GATT_OK){
            ESP_LOGE(TAG, "open failed, status %d", p_data->open.status);
            break;
        }
        ESP_LOGI(TAG, "open success");
        break;

    case ESP_GATTC_DIS_SRVC_CMPL_EVT:
        if (param->dis_srvc_cmpl.status != ESP_GATT_OK){
            ESP_LOGE(TAG, "discover service failed, status %d", param->dis_srvc_cmpl.status);
            break;
        }
        ESP_LOGI(TAG, "discover service complete conn_id %d", param->dis_srvc_cmpl.conn_id);
        
        // 获取服务信息
        esp_ble_gattc_search_service(gattc_if, param->cfg_mtu.conn_id, &remote_filter_service_uuid);
        break;

    case ESP_GATTC_SEARCH_RES_EVT: {
        ESP_LOGI(TAG, "SEARCH RES: conn_id = %x is primary service %d", p_data->search_res.conn_id, p_data->search_res.is_primary);
        ESP_LOGI(TAG, "start handle %d end handle %d current handle value %d", p_data->search_res.start_handle, p_data->search_res.end_handle, p_data->search_res.srvc_id.inst_id);
        
        if (p_data->search_res.srvc_id.uuid.len == ESP_UUID_LEN_16 && p_data->search_res.srvc_id.uuid.uuid.uuid16 == REMOTE_SERVICE_UUID) {
            ESP_LOGI(TAG, "service found");
            get_server = true;
            gl_profile_tab[PROFILE_A_APP_ID].service_start_handle = p_data->search_res.start_handle;
            gl_profile_tab[PROFILE_A_APP_ID].service_end_handle = p_data->search_res.end_handle;
            ESP_LOGI(TAG, "UUID16: %x", p_data->search_res.srvc_id.uuid.uuid.uuid16);
        }
        break;
    }

    case ESP_GATTC_SEARCH_CMPL_EVT:
        if (p_data->search_cmpl.status != ESP_GATT_OK){
            ESP_LOGE(TAG, "search service failed, error status = %x", p_data->search_cmpl.status);
            break;
        }
        
        if(p_data->search_cmpl.searched_service_source == ESP_GATT_SERVICE_FROM_REMOTE_DEVICE) {
            ESP_LOGI(TAG, "Get service information from remote device");
        } else if (p_data->search_cmpl.searched_service_source == ESP_GATT_SERVICE_FROM_NVS_FLASH) {
            ESP_LOGI(TAG, "Get service information from flash");
        } else {
            ESP_LOGI(TAG, "unknown service source");
        }
        
        ESP_LOGI(TAG, "ESP_GATTC_SEARCH_CMPL_EVT");
        if (get_server){
            uint16_t count = 0;
            esp_gatt_status_t status = esp_ble_gattc_get_attr_count( gattc_if,
                                                                     p_data->search_cmpl.conn_id,
                                                                     ESP_GATT_DB_CHARACTERISTIC,
                                                                     gl_profile_tab[PROFILE_A_APP_ID].service_start_handle,
                                                                     gl_profile_tab[PROFILE_A_APP_ID].service_end_handle,
                                                                     INVALID_HANDLE,
                                                                     &count);
            if (status != ESP_GATT_OK){
                ESP_LOGE(TAG, "esp_ble_gattc_get_attr_count error");
            }

            if (count > 0){
                char_elem_result = (esp_gattc_char_elem_t *)malloc(sizeof(esp_gattc_char_elem_t) * count);
                if (!char_elem_result){
                    ESP_LOGE(TAG, "gattc no mem");
                } else {
                    status = esp_ble_gattc_get_char_by_uuid( gattc_if,
                                                             p_data->search_cmpl.conn_id,
                                                             gl_profile_tab[PROFILE_A_APP_ID].service_start_handle,
                                                             gl_profile_tab[PROFILE_A_APP_ID].service_end_handle,
                                                             remote_filter_char_uuid,
                                                             char_elem_result,
                                                             &count);
                    if (status != ESP_GATT_OK){
                        ESP_LOGE(TAG, "esp_ble_gattc_get_char_by_uuid error");
                    }

                    if (count > 0 && (char_elem_result[0].properties & ESP_GATT_CHAR_PROP_BIT_NOTIFY)){
                        gl_profile_tab[PROFILE_A_APP_ID].char_handle = char_elem_result[0].char_handle;
                        esp_ble_gattc_register_for_notify (gattc_if, gl_profile_tab[PROFILE_A_APP_ID].remote_bda, char_elem_result[0].char_handle);
                    }
                }
                free(char_elem_result);
            } else {
                ESP_LOGE(TAG, "no char found");
            }
        }
        break;

    case ESP_GATTC_REG_FOR_NOTIFY_EVT: {
        ESP_LOGI(TAG, "ESP_GATTC_REG_FOR_NOTIFY_EVT");
        if (p_data->reg_for_notify.status != ESP_GATT_OK){
            ESP_LOGE(TAG, "REG FOR NOTIFY failed: error status = %d", p_data->reg_for_notify.status);
        } else {
            uint16_t count = 0;
            uint16_t notify_en = 1;
            esp_gatt_status_t ret_status = esp_ble_gattc_get_attr_count( gattc_if,
                                                                         gl_profile_tab[PROFILE_A_APP_ID].conn_id,
                                                                         ESP_GATT_DB_DESCRIPTOR,
                                                                         gl_profile_tab[PROFILE_A_APP_ID].service_start_handle,
                                                                         gl_profile_tab[PROFILE_A_APP_ID].service_end_handle,
                                                                         gl_profile_tab[PROFILE_A_APP_ID].char_handle,
                                                                         &count);
            if (ret_status != ESP_GATT_OK){
                ESP_LOGE(TAG, "esp_ble_gattc_get_attr_count error");
            }
            if (count > 0){
                descr_elem_result = malloc(sizeof(esp_gattc_descr_elem_t) * count);
                if (!descr_elem_result){
                    ESP_LOGE(TAG, "malloc error, gattc no mem");
                } else {
                    ret_status = esp_ble_gattc_get_descr_by_char_handle( gattc_if,
                                                                         gl_profile_tab[PROFILE_A_APP_ID].conn_id,
                                                                         p_data->reg_for_notify.handle,
                                                                         notify_descr_uuid,
                                                                         descr_elem_result,
                                                                         &count);
                    if (ret_status != ESP_GATT_OK){
                        ESP_LOGE(TAG, "esp_ble_gattc_get_descr_by_char_handle error");
                    }

                    if (count > 0 && descr_elem_result[0].uuid.len == ESP_UUID_LEN_16 && descr_elem_result[0].uuid.uuid.uuid16 == ESP_GATT_UUID_CHAR_CLIENT_CONFIG){
                        ret_status = esp_ble_gattc_write_char_descr( gattc_if,
                                                                     gl_profile_tab[PROFILE_A_APP_ID].conn_id,
                                                                     descr_elem_result[0].handle,
                                                                     sizeof(notify_en),
                                                                     (uint8_t *)&notify_en,
                                                                     ESP_GATT_WRITE_TYPE_RSP,
                                                                     ESP_GATT_AUTH_REQ_NONE);
                    }

                    if (ret_status != ESP_GATT_OK){
                        ESP_LOGE(TAG, "esp_ble_gattc_write_char_descr error");
                    }

                    free(descr_elem_result);
                }
            }
            else{
                ESP_LOGE(TAG, "decsr not found");
            }

        }
        break;
    }

    case ESP_GATTC_NOTIFY_EVT:
        if (p_data->notify.is_notify){
            ESP_LOGI(TAG, "ESP_GATTC_NOTIFY_EVT, receive notify value:");
        } else {
            ESP_LOGI(TAG, "ESP_GATTC_NOTIFY_EVT, receive indicate value:");
        }
        esp_log_buffer_hex(TAG, p_data->notify.value, p_data->notify.value_len);
        break;

    case ESP_GATTC_WRITE_DESCR_EVT:
        if (p_data->write.status != ESP_GATT_OK){
            ESP_LOGE(TAG, "write descr failed, error status = %x", p_data->write.status);
            break;
        }
        ESP_LOGI(TAG, "write descr success ");
        break;

    case ESP_GATTC_SRVC_CHG_EVT: {
        esp_bd_addr_t bda;
        memcpy(bda, p_data->srvc_chg.remote_bda, sizeof(esp_bd_addr_t));
        ESP_LOGI(TAG, "ESP_GATTC_SRVC_CHG_EVT, bd_addr:");
        esp_log_buffer_hex(TAG, bda, sizeof(esp_bd_addr_t));
        break;
    }

    case ESP_GATTC_DISCONNECT_EVT:
        connect = false;
        get_server = false;
        ESP_LOGI(TAG, "ESP_GATTC_DISCONNECT_EVT, reason = %d", p_data->disconnect.reason);
        break;

    default:
        break;
    }
}
```

---

> **蓝牙开发总结**：ESP32的BLE功能为IoT设备提供了低功耗、快速连接的无线通信能力。通过GAP进行设备发现和连接管理，通过GATT实现数据服务和交换。掌握服务器和客户端开发模式，能够构建完整的BLE应用生态系统。
