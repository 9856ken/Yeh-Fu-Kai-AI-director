import network
import time
import machine
import socket
import ssl
import ntptime

# =====================================
# Wi-Fi 設定
# =====================================

WIFI_SSID = "Ci-Xin"
WIFI_PASSWORD = "waldorf100"


# =====================================
# Gmail 設定
# =====================================

SMTP_SERVER = "smtp.gmail.com"
SMTP_PORT = 465

EMAIL_FROM = "9856ken@gmail.com"

# 這裡不是你的 Gmail 登入密碼
# 要填 Google App Password
EMAIL_PASSWORD = "khuw ywic mjtj lvoq "

EMAIL_TO = "9856ken@smail.ilc.edu.tw"


# =====================================
# PIR 設定
# =====================================

PIR_PIN = 27

pir = machine.Pin(PIR_PIN, machine.Pin.IN)


# =====================================
# 設定
# =====================================

# 下午4點以後才啟動
START_HOUR = 16

# 連續幾秒沒有偵測到人後寄信
# 測試可以先設定30秒
# 正式使用建議改成300秒 = 5分鐘
NO_PERSON_TIME = 300


# =====================================
# 連接 Wi-Fi
# =====================================

def connect_wifi():

    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)

    if not wlan.isconnected():

        print("正在連接 Wi-Fi...")

        wlan.connect(WIFI_SSID, WIFI_PASSWORD)

        while not wlan.isconnected():
            time.sleep(1)
            print(".", end="")

    print()
    print("Wi-Fi 連接成功")
    print("IP:", wlan.ifconfig()[0])

    return wlan


# =====================================
# 取得現在時間
# =====================================

def get_time():

    try:
        ntptime.settime()
        print("NTP 時間同步成功")
    except:
        print("NTP 時間同步失敗")


# =====================================
# 取得台灣時間
# UTC + 8
# =====================================

def taiwan_time():

    t = time.localtime(time.time() + 8 * 3600)

    return t


# =====================================
# 發送 Email
# =====================================

def send_email():

    print("開始發送 Email...")

    subject = "【智慧教室通知】下午4點後偵測不到人"

    body = """
智慧教室通知

現在時間已經超過下午4點。

ESP32 的紅外線人體感測器已經連續5分鐘沒有偵測到人體活動。

請確認教室是否已經沒有人。

ESP32 智慧教室系統
"""

    # 建立 Email
    message = (
        "From: ESP32 智慧教室 <" + EMAIL_FROM + ">\r\n"
        "To: " + EMAIL_TO + "\r\n"
        "Subject: " + subject + "\r\n"
        "Content-Type: text/plain; charset=utf-8\r\n"
        "\r\n"
        + body
    )

    # 建立 Socket
    addr = socket.getaddrinfo(
        SMTP_SERVER,
        SMTP_PORT
    )[0][-1]

    sock = socket.socket()

    try:

        print("連接 Gmail SMTP...")

        sock.connect(addr)

        # SSL
        ssock = ssl.wrap_socket(sock)

        print("Gmail SMTP 連接成功")

        # 讀取 Gmail 回覆
        def get_response():
            response = ssock.readline()
            print(response)
            return response

        get_response()

        # EHLO
        ssock.write(
            b"EHLO esp32\r\n"
        )

        get_response()

        # 登入
        import ubinascii

        auth_string = (
            "\0" +
            EMAIL_FROM +
            "\0" +
            EMAIL_PASSWORD
        )

        auth = ubinascii.b2a_base64(
            auth_string.encode()
        ).decode().strip()

        ssock.write(
            ("AUTH PLAIN " + auth + "\r\n").encode()
        )

        response = get_response()

        # 寄件者
        ssock.write(
            ("MAIL FROM:<" + EMAIL_FROM + ">\r\n").encode()
        )

        get_response()

        # 收件者
        ssock.write(
            ("RCPT TO:<" + EMAIL_TO + ">\r\n").encode()
        )

        get_response()

        # 開始寄信
        ssock.write(
            b"DATA\r\n"
        )

        get_response()

        ssock.write(
            message.encode("utf-8")
        )

        ssock.write(
            b"\r\n.\r\n"
        )

        response = get_response()

        # 結束
        ssock.write(
            b"QUIT\r\n"
        )

        get_response()

        ssock.close()

        print("Email 發送完成！")

        return True

    except Exception as e:

        print("Email 發送失敗")
        print(e)

        try:
            ssock.close()
        except:
            pass

        return False


# =====================================
# 主程式
# =====================================

print("==============================")
print("ESP32 智慧教室系統")
print("==============================")

connect_wifi()

get_time()

print("系統啟動")
print("下午4點後才會開始判斷")
print("目前設定：連續5分鐘沒有偵測到人")
print()


# 記錄最後一次偵測到人的時間
last_motion_time = time.time()

# 防止同一次無人狀態一直寄信
email_sent = False


while True:

    now = taiwan_time()

    hour = now[3]
    minute = now[4]
    second = now[5]

    # ---------------------------------
    # 下午4點以前
    # ---------------------------------

    if hour < START_HOUR:

        print(
            "目前時間 {:02d}:{:02d}:{:02d}，"
            "尚未到下午4點".format(
                hour,
                minute,
                second
            )
        )

        # 重置
        last_motion_time = time.time()
        email_sent = False

        time.sleep(10)

        continue


    # ---------------------------------
    # 下午4點以後
    # ---------------------------------

    sensor = pir.value()

    # PIR HIGH = 偵測到人體活動
    if sensor == 1:

        print(
            "{:02d}:{:02d}:{:02d} → 偵測到人".format(
                hour,
                minute,
                second
            )
        )

        # 更新最後偵測時間
        last_motion_time = time.time()

        # 有人了，允許下一次重新寄信
        email_sent = False

    else:

        no_person_time = (
            time.time() - last_motion_time
        )

        print(
            "{:02d}:{:02d}:{:02d} → "
            "沒有偵測到人 {:.0f} 秒".format(
                hour,
                minute,
                second,
                no_person_time
            )
        )

        # ---------------------------------
        # 連續5分鐘沒人
        # ---------------------------------

        if (
            no_person_time >= NO_PERSON_TIME
            and email_sent == False
        ):

            print("已經連續5分鐘沒有偵測到人")

            success = send_email()

            if success:

                email_sent = True

                print(
                    "通知已寄到：",
                    EMAIL_TO
                )

    time.sleep(1)
