# gralde环境配置



### gradle安装位置

![image-20260112171408944](Android.assets/image-20260112171408944.png)

### setting.gradle.kts

```
pluginManagement {
    repositories {
        // 阿里云镜像优先，适配Gradle 8.7
        maven("https://maven.aliyun.com/repository/google")
        maven("https://maven.aliyun.com/repository/gradle-plugin")
        maven("https://maven.aliyun.com/repository/public")
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        maven("https://maven.aliyun.com/repository/google")
        maven("https://maven.aliyun.com/repository/public")
        mavenCentral()
    }
}

rootProject.name = "CAR"
include(":app")
```

### build.gradile.kts (app)

```
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    // 替换成你项目的实际包名（比如com.xxx.car）
    namespace = "com.example.car"
    compileSdk = 34 // AGP 8.4.0适配的推荐版本（Gradle 8.7要求）

    defaultConfig {
        applicationId = "com.example.car" // 替换成你的应用ID
        minSdk = 21
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17 // Gradle 8.7要求Java 17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.12.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")

    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
}
```

或者用下面的这个

```
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
}

android {
    namespace = "com.esp_app"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.esp_app"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
    }
}

dependencies {

    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.appcompat)
    implementation(libs.material)
    implementation(libs.androidx.activity)
    implementation(libs.androidx.constraintlayout)
    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)
}
```

### build.gradile.kts (project)

```
// Gradle 8.7推荐用plugins块声明插件（替代旧的buildscript），无版本冲突
plugins {
    id("com.android.application") version "8.4.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.0" apply false
}

// 修复buildDir弃用，适配Gradle 8.x
tasks.register("clean", Delete::class) {
    delete(layout.buildDirectory.get().asFile)
}
```



# WIFI连接(manager方法(已过时))

```
package com.esp_app;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;

import androidx.activity.EdgeToEdge;
import androidx.core.graphics.Insets;
import androidx.core.view.ViewCompat;
import androidx.core.view.WindowInsetsCompat;

import android.Manifest;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.pm.PackageManager;
import android.net.ConnectivityManager;
import android.net.Network;
import android.net.NetworkCapabilities;
import android.net.NetworkRequest;
import android.net.wifi.WifiConfiguration;
import android.net.wifi.WifiManager;
import android.net.wifi.WifiNetworkSpecifier;
import android.os.Build;
import android.os.Bundle;
import android.text.TextUtils;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;
import android.widget.ProgressBar;
import android.widget.TextView;
import android.widget.Toast;

import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class MainActivity extends AppCompatActivity {

    private EditText ssidInput, passwordInput, ipInput, portInput;
    private Button connectButton;
    private ProgressBar progressBar;
    private TextView statusText;
    private WifiManager wifiManager;
    private ConnectivityManager connectivityManager;
    private ExecutorService executorService;

    // SharedPreferences 键名
    private static final String PREFS_NAME = "WifiConfigPrefs";
    private static final String KEY_SSID = "ssid";
    private static final String KEY_PASSWORD = "password";
    private static final String KEY_IP = "ip";
    private static final String KEY_PORT = "port";

    private static final int PERMISSIONS_REQUEST_CODE = 100;
    private static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        EdgeToEdge.enable(this);
        setContentView(R.layout.activity_main);
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main1), (v, insets) -> {
            Insets systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars());
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom);
            return insets;
        });

        // 初始化线程池
        executorService = Executors.newSingleThreadExecutor();

        // 初始化视图
        ipInput = findViewById(R.id.Ip_input);
        portInput = findViewById(R.id.Port_input);
        ssidInput = findViewById(R.id.ssid_input);
        passwordInput = findViewById(R.id.password_input);
        connectButton = findViewById(R.id.ack_button);
        progressBar = findViewById(R.id.progressBar);
        statusText = findViewById(R.id.status_text);
        // 初始化系统服务
        wifiManager = (WifiManager) getApplicationContext().getSystemService(Context.WIFI_SERVICE);
        connectivityManager = (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);

        // 加载保存的配置信息
        loadSavedConfig();

        // 检查并请求权限
        checkPermissions();

        // 设置连接按钮点击事件
        connectButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                attemptConnection();
            }
        });

    }

    // 加载保存的配置信息
    private void loadSavedConfig() {
        SharedPreferences prefs = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
        ssidInput.setText(prefs.getString(KEY_SSID, ""));
        passwordInput.setText(prefs.getString(KEY_PASSWORD, ""));
        ipInput.setText(prefs.getString(KEY_IP, ""));
        portInput.setText(prefs.getString(KEY_PORT, "8888")); // 默认端口8888
    }
    // 保存配置信息
    private void saveConfig(String ssid, String password, String ip, String port) {
        SharedPreferences.Editor editor = getSharedPreferences(PREFS_NAME, MODE_PRIVATE).edit();
        editor.putString(KEY_SSID, ssid);
        editor.putString(KEY_PASSWORD, password);
        editor.putString(KEY_IP, ip);
        editor.putString(KEY_PORT, port);
        editor.apply();
    }


    private void checkPermissions() {
        String[] permissions;

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            permissions = new String[]{
                    Manifest.permission.ACCESS_WIFI_STATE,
                    Manifest.permission.CHANGE_WIFI_STATE,
                    Manifest.permission.ACCESS_NETWORK_STATE,
                    Manifest.permission.ACCESS_FINE_LOCATION
            };
        } else {
            permissions = new String[]{
                    Manifest.permission.ACCESS_WIFI_STATE,
                    Manifest.permission.CHANGE_WIFI_STATE,
                    Manifest.permission.ACCESS_NETWORK_STATE
            };
        }

        boolean allPermissionsGranted = true;
        for (String permission : permissions) {
            if (ActivityCompat.checkSelfPermission(this, permission) != PackageManager.PERMISSION_GRANTED) {
                allPermissionsGranted = false;
                break;
            }
        }

        if (!allPermissionsGranted) {
            ActivityCompat.requestPermissions(this, permissions, PERMISSIONS_REQUEST_CODE);
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == PERMISSIONS_REQUEST_CODE) {
            boolean allGranted = true;
            for (int grantResult : grantResults) {
                if (grantResult != PackageManager.PERMISSION_GRANTED) {
                    allGranted = false;
                    break;
                }
            }

            if (!allGranted) {
                Toast.makeText(this, "需要所有权限才能连接WiFi", Toast.LENGTH_SHORT).show();
            }
        }
    }

    private void attemptConnection() {
        String ssid = ssidInput.getText().toString().trim();
        String password = passwordInput.getText().toString();

        String ip = ipInput.getText().toString().trim();
        String port_char = portInput.getText().toString().trim();

        int port = 8888;
        try
        {
            if (TextUtils.isEmpty(port_char)) {
                // 如果为空，使用默认端口
                port = 8888;
                Toast.makeText(this, "使用默认端口: 8888", Toast.LENGTH_SHORT).show();
            } else {
                // 尝试转换为整数
                port = Integer.parseInt(port_char);

                // 验证端口范围 (1-65535)
                if (port < 1 || port > 65535) {
                    Toast.makeText(this, "端口号必须在1-65535之间，使用默认端口8888", Toast.LENGTH_SHORT).show();
                    port = 8888;
                }
            }
            } catch (NumberFormatException e) {
                // 处理数字格式异常
                e.printStackTrace();
                Toast.makeText(this, "端口号格式错误，使用默认端口8888", Toast.LENGTH_SHORT).show();
                port = 8888; // 使用默认端口
        }

        if (TextUtils.isEmpty(password)) {
            Toast.makeText(this, "请输入password", Toast.LENGTH_SHORT).show();
            return;
        } else if (TextUtils.isEmpty(ssid)) {
            Toast.makeText(this, "请输入ssid", Toast.LENGTH_SHORT).show();
            return;
        }

        // 显示进度条
        progressBar.setVisibility(View.VISIBLE);
        statusText.setText("正在连接...");
        connectButton.setEnabled(false);

        try {
            // 检查Android版本，使用不同的连接方式
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                // Android 10+ 使用新的API
                connectWithNewApi(ssid, password, ip, port);
            } else {
                // 旧版本Android使用传统方式
                connectWithLegacyApi(ssid, password);
            }
        } catch (Exception e) {
            Log.e(TAG, "连接过程中发生错误", e);
            runOnUiThread(() -> {
                progressBar.setVisibility(View.GONE);
                statusText.setText("连接错误");
                Toast.makeText(MainActivity.this, "连接过程中发生错误: " + e.getMessage(), Toast.LENGTH_SHORT).show();
                connectButton.setEnabled(true);
            });
        }
    }

    @androidx.annotation.RequiresApi(api = Build.VERSION_CODES.Q)
    private void connectWithNewApi(String ssid, String password,String ip ,int port) {
        try {
            WifiNetworkSpecifier.Builder builder = new WifiNetworkSpecifier.Builder();
            builder.setSsid(ssid);
            builder.setWpa2Passphrase(password);

            WifiNetworkSpecifier wifiNetworkSpecifier = builder.build();

            NetworkRequest.Builder requestBuilder = new NetworkRequest.Builder();
            requestBuilder.addTransportType(NetworkCapabilities.TRANSPORT_WIFI);
            requestBuilder.removeCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET);
            requestBuilder.addCapability(NetworkCapabilities.NET_CAPABILITY_NOT_RESTRICTED);
            requestBuilder.addCapability(NetworkCapabilities.NET_CAPABILITY_TRUSTED);
            requestBuilder.setNetworkSpecifier(wifiNetworkSpecifier);

            NetworkRequest request = requestBuilder.build();

            ConnectivityManager.NetworkCallback networkCallback = new ConnectivityManager.NetworkCallback() {
                @Override
                public void onAvailable(@NonNull Network network) {
                    super.onAvailable(network);
                    runOnUiThread(() -> {
                        progressBar.setVisibility(View.GONE);
                        statusText.setText("连接成功!");
                        Toast.makeText(MainActivity.this, "已成功连接到 " + ssid, Toast.LENGTH_SHORT).show();

                        // 保存当前配置
                        saveConfig(ssid, password, ip, String.valueOf(port));

                        Intent intent = new Intent(MainActivity.this, ShowActivity.class);
                        intent.putExtra("IP", ip);
                        intent.putExtra("PORT", port);
                        // 传递信息
                        startActivity(intent);
                        connectButton.setEnabled(true);
                    });
                }

                @Override
                public void onUnavailable() {
                    super.onUnavailable();
                    runOnUiThread(() -> {
                        progressBar.setVisibility(View.GONE);
                        statusText.setText("连接失败");
                        Toast.makeText(MainActivity.this, "连接失败，请检查SSID和密码", Toast.LENGTH_SHORT).show();
                        connectButton.setEnabled(true);
                    });
                }

                @Override
                public void onLost(@NonNull Network network) {
                    super.onLost(network);
                    runOnUiThread(() -> {
                        progressBar.setVisibility(View.GONE);
                        statusText.setText("连接丢失");
                        Toast.makeText(MainActivity.this, "连接已丢失", Toast.LENGTH_SHORT).show();
                        connectButton.setEnabled(true);
                    });
                }
            };

            connectivityManager.requestNetwork(request, networkCallback);
        } catch (Exception e) {
            Log.e(TAG, "使用新API连接时发生错误", e);
            runOnUiThread(() -> {
                progressBar.setVisibility(View.GONE);
                statusText.setText("连接错误");
                Toast.makeText(MainActivity.this, "连接过程中发生错误: " + e.getMessage(), Toast.LENGTH_SHORT).show();
                connectButton.setEnabled(true);
            });
        }
    }

    private void connectWithLegacyApi(String ssid, String password) {
        executorService.execute(() -> {
            try {
                // 传统WiFi连接方式
                WifiConfiguration wifiConfig = new WifiConfiguration();
                wifiConfig.SSID = String.format("\"%s\"", ssid);
                wifiConfig.preSharedKey = String.format("\"%s\"", password);

                // 设置其他必要的配置
                wifiConfig.allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.OPEN);
                wifiConfig.allowedProtocols.set(WifiConfiguration.Protocol.RSN);
                wifiConfig.allowedProtocols.set(WifiConfiguration.Protocol.WPA);
                wifiConfig.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.WPA_PSK);
                wifiConfig.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.CCMP);
                wifiConfig.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.TKIP);
                wifiConfig.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP);
                wifiConfig.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.TKIP);
                wifiConfig.status = WifiConfiguration.Status.ENABLED;

                // 确保WiFi已启用
                if (!wifiManager.isWifiEnabled()) {
                    wifiManager.setWifiEnabled(true);
                    // 等待WiFi启用
                    Thread.sleep(2000);
                }

                // 移除任何现有的相同SSID的网络
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M
                        && ActivityCompat.checkSelfPermission(MainActivity.this, Manifest.permission.ACCESS_FINE_LOCATION)
                        != PackageManager.PERMISSION_GRANTED) {
                    runOnUiThread(() -> {
                        progressBar.setVisibility(View.GONE);
                        Toast.makeText(MainActivity.this, "需要定位权限才能管理WiFi网络", Toast.LENGTH_SHORT).show();
                        connectButton.setEnabled(true);
                    });
                    return;
                }
                List<WifiConfiguration> existingConfigs = wifiManager.getConfiguredNetworks();
                if (existingConfigs != null) {
                    for (WifiConfiguration existingConfig : existingConfigs) {
                        if (existingConfig.SSID != null && existingConfig.SSID.equals("\"" + ssid + "\"")) {
                            wifiManager.removeNetwork(existingConfig.networkId);
                            wifiManager.saveConfiguration();
                        }
                    }
                }

                int netId = wifiManager.addNetwork(wifiConfig);
                if (netId == -1) {
                    runOnUiThread(() -> {
                        progressBar.setVisibility(View.GONE);
                        statusText.setText("配置失败");
                        Toast.makeText(MainActivity.this, "网络配置失败", Toast.LENGTH_SHORT).show();
                        connectButton.setEnabled(true);
                    });
                    return;
                }

                // 断开当前连接
                wifiManager.disconnect();

                // 启用新网络
                boolean enabled = wifiManager.enableNetwork(netId, true);

                // 重新连接
                boolean reconnected = wifiManager.reconnect();

                // 等待连接结果
                Thread.sleep(5000); // 等待5秒让连接建立

                runOnUiThread(() -> {
                    progressBar.setVisibility(View.GONE);
                    if (enabled && reconnected) {
                        statusText.setText("连接成功!");
                        Toast.makeText(MainActivity.this, "已成功连接到 " + ssid, Toast.LENGTH_SHORT).show();
                        Intent intent = new Intent(MainActivity.this, ShowActivity.class);
                        intent.putExtra("SSID", ssid);
                        startActivity(intent);
                    } else {
                        statusText.setText("连接失败");
                        Toast.makeText(MainActivity.this, "连接失败，请检查SSID和密码", Toast.LENGTH_SHORT).show();
                    }
                    connectButton.setEnabled(true);
                });
            } catch (Exception e) {
                Log.e(TAG, "使用传统API连接时发生错误", e);
                runOnUiThread(() -> {
                    progressBar.setVisibility(View.GONE);
                    statusText.setText("连接错误");
                    Toast.makeText(MainActivity.this, "连接过程中发生错误: " + e.getMessage(), Toast.LENGTH_SHORT).show();
                    connectButton.setEnabled(true);
                });
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (executorService != null) {
            executorService.shutdown();
        }
    }
}
```

| 控件        | 作用            |
| ----------- | --------------- |
| SSID        | Wi-Fi 名        |
| Password    | Wi-Fi 密码      |
| IP          | ESP / 服务器 IP |
| Port        | 通信端口        |
| Button      | 点击开始连接    |
| ProgressBar | 显示连接过程    |
| TextView    | 显示连接状态    |

```
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
```

# WIFI连接(WifiNetworkSpecifier)

package com.esp_app;

import android.Manifest;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.pm.PackageManager;
import android.net.ConnectivityManager;
import android.net.Network;
import android.net.NetworkCapabilities;
import android.net.NetworkRequest;
import android.net.wifi.WifiNetworkSpecifier;
import android.os.Build;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;
import android.widget.ProgressBar;
import android.widget.TextView;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.annotation.RequiresApi;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.app.ActivityCompat;

public class MainActivity extends AppCompatActivity {

    private EditText ssidInput, passwordInput, ipInput, portInput;
    private Button connectButton;
    private ProgressBar progressBar;
    private TextView statusText;
    
    private ConnectivityManager connectivityManager;
    
    private static final int PERMISSIONS_REQUEST_CODE = 100;
    
    private static final String PREFS_NAME = "WifiConfigPrefs";
    private static final String KEY_SSID = "ssid";
    private static final String KEY_PASSWORD = "password";
    private static final String KEY_IP = "ip";
    private static final String KEY_PORT = "port";
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    
        ssidInput = findViewById(R.id.ssid_input);
        passwordInput = findViewById(R.id.password_input);
        ipInput = findViewById(R.id.Ip_input);
        portInput = findViewById(R.id.Port_input);
        connectButton = findViewById(R.id.ack_button);
        progressBar = findViewById(R.id.progressBar);
        statusText = findViewById(R.id.status_text);
    
        connectivityManager =
                (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
    
        loadSavedConfig();
        checkPermission();
    
        connectButton.setOnClickListener(v -> attemptConnection());
    }
    
    // ================= 权限 =================
    
    private void checkPermission() {
        if (ActivityCompat.checkSelfPermission(
                this,
                Manifest.permission.ACCESS_FINE_LOCATION)
                != PackageManager.PERMISSION_GRANTED) {
    
            ActivityCompat.requestPermissions(
                    this,
                    new String[]{Manifest.permission.ACCESS_FINE_LOCATION},
                    PERMISSIONS_REQUEST_CODE
            );
        }
    }
    
    @Override
    public void onRequestPermissionsResult(
            int requestCode,
            @NonNull String[] permissions,
            @NonNull int[] grantResults) {
    
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    
        if (requestCode == PERMISSIONS_REQUEST_CODE &&
                (grantResults.length == 0 ||
                        grantResults[0] != PackageManager.PERMISSION_GRANTED)) {
    
            Toast.makeText(this,
                    "需要定位权限才能连接 WiFi",
                    Toast.LENGTH_LONG).show();
        }
    }
    
    // ================= 连接 =================
    
    private void attemptConnection() {
    
        String ssid = ssidInput.getText().toString().trim();
        String password = passwordInput.getText().toString();
        String ip = ipInput.getText().toString().trim();
        String portStr = portInput.getText().toString().trim();
    
        if (TextUtils.isEmpty(ssid) || TextUtils.isEmpty(password)) {
            Toast.makeText(this, "SSID 或密码不能为空", Toast.LENGTH_SHORT).show();
            return;
        }
    
        int port = 8888;
        try {
            if (!TextUtils.isEmpty(portStr)) {
                port = Integer.parseInt(portStr);
            }
        } catch (Exception ignored) {
            port = 8888;
        }
    
        progressBar.setVisibility(View.VISIBLE);
        statusText.setText("正在连接...");
        connectButton.setEnabled(false);
    
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            connectWithWifiSpecifier(ssid, password, ip, port);
        } else {
            Toast.makeText(this,
                    "Android 10 以下不支持该方式",
                    Toast.LENGTH_LONG).show();
        }
    }
    
    // ================= WifiNetworkSpecifier =================
    
    @RequiresApi(api = Build.VERSION_CODES.Q)
    private void connectWithWifiSpecifier(
            String ssid,
            String password,
            String ip,
            int port) {
    
        WifiNetworkSpecifier specifier =
                new WifiNetworkSpecifier.Builder()
                        .setSsid(ssid)
                        .setWpa2Passphrase(password)
                        .build();
    
        NetworkRequest request =
                new NetworkRequest.Builder()
                        .addTransportType(NetworkCapabilities.TRANSPORT_WIFI)
                        .setNetworkSpecifier(specifier)
                        .build();
    
        ConnectivityManager.NetworkCallback callback =
                new ConnectivityManager.NetworkCallback() {
    
                    @Override
                    public void onAvailable(@NonNull Network network) {
                        super.onAvailable(network);
    
                        connectivityManager.bindProcessToNetwork(network);
    
                        runOnUiThread(() -> {
                            progressBar.setVisibility(View.GONE);
                            statusText.setText("连接成功");
    
                            saveConfig(ssid, password, ip, String.valueOf(port));
    
                            Intent intent =
                                    new Intent(MainActivity.this, ShowActivity.class);
                            intent.putExtra("IP", ip);
                            intent.putExtra("PORT", port);
                            startActivity(intent);
    
                            connectButton.setEnabled(true);
                        });
                    }
    
                    @Override
                    public void onUnavailable() {
                        runOnUiThread(() -> {
                            progressBar.setVisibility(View.GONE);
                            statusText.setText("连接失败");
                            connectButton.setEnabled(true);
                        });
                    }
    
                    @Override
                    public void onLost(@NonNull Network network) {
                        runOnUiThread(() ->
                                statusText.setText("WiFi 已断开"));
                    }
                };
    
        connectivityManager.requestNetwork(request, callback);
    }
    
    // ================= 配置保存 =================
    
    private void saveConfig(String ssid, String password, String ip, String port) {
        SharedPreferences.Editor editor =
                getSharedPreferences(PREFS_NAME, MODE_PRIVATE).edit();
        editor.putString(KEY_SSID, ssid);
        editor.putString(KEY_PASSWORD, password);
        editor.putString(KEY_IP, ip);
        editor.putString(KEY_PORT, port);
        editor.apply();
    }
    
    private void loadSavedConfig() {
        SharedPreferences prefs =
                getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
        ssidInput.setText(prefs.getString(KEY_SSID, ""));
        passwordInput.setText(prefs.getString(KEY_PASSWORD, ""));
        ipInput.setText(prefs.getString(KEY_IP, ""));
        portInput.setText(prefs.getString(KEY_PORT, "8888"));
    	}
    }
 

```
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```



#  获取地址端口再连接+接受数据

```
// 获取传递过来的数据并连接ESP
Intent intent = getIntent();
if (intent != null) {
    String ip = intent.getStringExtra("IP");
    int port = intent.getIntExtra("PORT", 8888);

    // 连接ESP8266并开始接收数据
    connectToESPWithRetry(ip, port, 3);
} else {
    Toast.makeText(this, "未收到连接参数", Toast.LENGTH_SHORT).show();
    finish();
}
```

```
private void connectToESPWithRetry(String ESP_IP, int ESP_PORT, int retryCount) {
    if (retryCount <= 0) {
        handler.post(new Runnable() {
            @Override
            public void run() {
                Toast.makeText(ShowActivity.this,
                        "连接失败，已达到最大重试次数",
                        Toast.LENGTH_SHORT).show();
                if (!isStoppedByUser) {
                    finish();
                }
            }
        });
        return;
    }

    handler.post(new Runnable() {
        @Override
        public void run() {
            Toast.makeText(ShowActivity.this,
                    "尝试连接到: " + ESP_IP + ":" + ESP_PORT,
                    Toast.LENGTH_SHORT).show();
        }
    });

    executorService.execute(new Runnable() {
        @Override
        public void run() {
            try {
                // 建立Socket连接
                socket = new Socket();
                socket.connect(new InetSocketAddress(ESP_IP, ESP_PORT), 5000);
                reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                isConnected = true;

                // 更新UI显示连接成功
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(ShowActivity.this, "已连接到ESP8266", Toast.LENGTH_SHORT).show();
                    }
                });

                // 持续接收数据
                while (isConnected && !isStoppedByUser) {
                    String data = reader.readLine();
                    if (data != null) {
                        parseAndUpdateData(data);
                    } else {
                        // 服务器关闭连接
                        isConnected = false;
                        if (!isStoppedByUser) {
                            handler.post(new Runnable() {
                                @Override
                                public void run() {
                                    Toast.makeText(ShowActivity.this,
                                            "服务器关闭连接，正在重试 (" + (retryCount-1) + "次剩余)",
                                            Toast.LENGTH_LONG).show();
                                }
                            });

                            // 等待一段时间后重试
                            try {
                                Thread.sleep(2000);
                            } catch (InterruptedException ie) {
                                ie.printStackTrace();
                            }

                            connectToESPWithRetry(ESP_IP, ESP_PORT, retryCount - 1);
                            return;
                        }
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
                if (!isStoppedByUser) {
                    final String errorMsg = e.getMessage();
                    handler.post(new Runnable() {
                        @Override
                        public void run() {
                            Toast.makeText(ShowActivity.this,
                                    "连接失败: " + errorMsg + ", 正在重试 (" + (retryCount-1) + "次剩余)",
                                    Toast.LENGTH_LONG).show();
                        }
                    });

                    // 等待一段时间后重试
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException ie) {
                        ie.printStackTrace();
                    }

                    connectToESPWithRetry(ESP_IP, ESP_PORT, retryCount - 1);
                }
            } finally {
                closeConnection();
            }
        }
    });
}
```

# 发送数据

```
private void sendStopCommand() {
    executorService.execute(new Runnable() {
        @Override
        public void run() {
            try {
                if (socket != null && !socket.isClosed()) {
                    // 发送停止命令
                    socket.getOutputStream().write("STOP\n".getBytes());
                    socket.getOutputStream().flush();

                    // 等待一段时间，让服务器处理命令
                    Thread.sleep(1000);

                    // 然后优雅地关闭连接
                    closeConnection();

                    handler.post(new Runnable() {
                        @Override
                        public void run() {
                            Toast.makeText(ShowActivity.this, "已发送停止命令并断开连接", Toast.LENGTH_SHORT).show();
                        }
                    });
                }
            } catch (IOException e) {
                e.printStackTrace();
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(ShowActivity.this, "发送停止命令失败", Toast.LENGTH_SHORT).show();
                        endAckButton.setEnabled(true);
                        endAckButton.setText("关闭");
                    }
                });
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
}
```

# XML

