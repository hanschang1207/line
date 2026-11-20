import json
import os
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path
from urllib.error import HTTPError, URLError
from urllib.parse import parse_qs, urlencode, urlsplit
from urllib.request import Request, urlopen


ROOT = Path(__file__).resolve().parent
LINE_VERIFY_URL = "https://api.line.me/oauth2/v2.1/verify"
LINE_PUSH_URL = "https://api.line.me/v2/bot/message/push"
MAX_REQUEST_BYTES = 16_384
MAX_QUESTION_LENGTH = 160
MAX_ANSWER_LENGTH = 4_000
MAX_LINE_MESSAGE_LENGTH = 5_000


class BridgeError(Exception):
    def __init__(self, status, message):
        super().__init__(message)
        self.status = status


def allowed_origins():
    configured = os.environ.get(
        "CORS_ORIGINS",
        "http://localhost:8000,http://127.0.0.1:8000",
    )
    return {origin.strip().rstrip("/") for origin in configured.split(",") if origin.strip()}


def line_configuration():
    login_channel_id = os.environ.get("LINE_LOGIN_CHANNEL_ID", "").strip()
    access_token = os.environ.get("LINE_CHANNEL_ACCESS_TOKEN", "").strip()
    liff_id = os.environ.get("LINE_LIFF_ID", "").strip()
    return {
        "loginChannelId": login_channel_id,
        "accessToken": access_token,
        "liffId": liff_id,
        "enabled": bool(login_channel_id and access_token and liff_id),
    }


def verify_line_id_token(id_token, login_channel_id):
    body = urlencode({"id_token": id_token, "client_id": login_channel_id}).encode("ascii")
    request = Request(
        LINE_VERIFY_URL,
        data=body,
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        method="POST",
    )
    try:
        with urlopen(request, timeout=10) as response:
            claims = json.loads(response.read().decode("utf-8"))
    except HTTPError as error:
        raise BridgeError(401, "LINE 登入憑證無效或已過期，請重新登入。") from error
    except (URLError, TimeoutError) as error:
        raise BridgeError(502, "目前無法向 LINE 驗證登入憑證，請稍後再試。") from error
    except (UnicodeDecodeError, json.JSONDecodeError) as error:
        raise BridgeError(502, "LINE 登入驗證服務回傳無法辨識的資料。") from error

    if claims.get("aud") != login_channel_id or not isinstance(claims.get("sub"), str):
        raise BridgeError(401, "LINE 登入憑證與此服務的登入頻道不相符。")
    return claims["sub"]


def push_line_message(user_id, message, access_token):
    body = json.dumps({
        "to": user_id,
        "messages": [{"type": "text", "text": message}],
    }, ensure_ascii=False).encode("utf-8")
    request = Request(
        LINE_PUSH_URL,
        data=body,
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json",
        },
        method="POST",
    )
    try:
        with urlopen(request, timeout=10) as response:
            response.read()
    except HTTPError as error:
        print(f"LINE push failed with HTTP {error.code}.", flush=True)
        raise BridgeError(
            502,
            "LINE 訊息傳送失敗，請確認官方帳號設定、Access token 與好友關係。",
        ) from error
    except (URLError, TimeoutError) as error:
        raise BridgeError(502, "目前無法連線到 LINE 訊息服務，請稍後再試。") from error


class LineBridgeHandler(BaseHTTPRequestHandler):
    server_version = "LineBridge/1.0"

    def _send_json(self, status, payload):
        body = json.dumps(payload, ensure_ascii=False).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Cache-Control", "no-store")
        self._send_cors_headers()
        self.end_headers()
        self.wfile.write(body)

    def _send_cors_headers(self):
        origin = self.headers.get("Origin")
        if origin and origin.rstrip("/") in allowed_origins():
            self.send_header("Access-Control-Allow-Origin", origin)
            self.send_header("Vary", "Origin")

    def _check_origin(self):
        origin = self.headers.get("Origin")
        if origin and origin.rstrip("/") not in allowed_origins():
            raise BridgeError(403, "此網站來源未列入伺服器允許清單。")

    def do_OPTIONS(self):
        try:
            self._check_origin()
        except BridgeError as error:
            self._send_json(error.status, {"error": str(error)})
            return
        self.send_response(204)
        self.send_header("Content-Length", "0")
        self.send_header("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
        self.send_header("Access-Control-Allow-Headers", "Content-Type")
        self.send_header("Access-Control-Max-Age", "600")
        self._send_cors_headers()
        self.end_headers()

    def do_GET(self):
        path = urlsplit(self.path).path
        if path == "/api/config":
            config = line_configuration()
            self._send_json(200, {
                "enabled": config["enabled"],
                "liffId": config["liffId"],
                "error": None if config["enabled"] else "伺服器尚未設定完整的 LINE 環境變數。",
            })
            return
        if path == "/health":
            self._send_json(200, {"ok": True})
            return
        if path in ("/", "/line-chat.html"):
            self._send_static_page("line-chat.html")
            return
        if path == "/night-sparrow-ordering.html":
            self._send_static_page()
            return
        self._send_json(404, {"error": "找不到此路徑。"})

    def do_POST(self):
        if urlsplit(self.path).path != "/api/line/push":
            self._send_json(404, {"error": "找不到此路徑。"})
            return
        try:
            self._check_origin()
            config = line_configuration()
            if not config["enabled"]:
                raise BridgeError(503, "伺服器尚未設定完整的 LINE 環境變數。")
            if self.headers.get_content_type() != "application/json":
                raise BridgeError(415, "請使用 application/json 傳送資料。")
            content_length = self.headers.get("Content-Length", "")
            if not content_length.isdigit() or int(content_length) > MAX_REQUEST_BYTES:
                raise BridgeError(413, "請求內容格式不正確或超過大小限制。")
            payload = json.loads(self.rfile.read(int(content_length)).decode("utf-8"))
            if not isinstance(payload, dict):
                raise BridgeError(400, "請求內容必須是 JSON 物件。")
            id_token = payload.get("idToken")
            question = payload.get("question")
            answer = payload.get("answer")
            if not isinstance(id_token, str) or not id_token or len(id_token) > 6_000:
                raise BridgeError(400, "缺少有效的 LINE 登入憑證。")
            if not isinstance(question, str) or not question.strip() or len(question) > MAX_QUESTION_LENGTH:
                raise BridgeError(400, "問題必須為 1 至 160 個字元。")
            if not isinstance(answer, str) or not answer.strip() or len(answer) > MAX_ANSWER_LENGTH:
                raise BridgeError(400, "回答必須為 1 至 4000 個字元。")

            user_id = verify_line_id_token(id_token, config["loginChannelId"])
            message = f"網站提問：{question.strip()}\n夜雀食堂小幫手：{answer.strip()}"
            if len(message) > MAX_LINE_MESSAGE_LENGTH:
                raise BridgeError(400, "組合後的訊息超過 LINE 文字訊息長度限制。")
            push_line_message(user_id, message, config["accessToken"])
        except BridgeError as error:
            self._send_json(error.status, {"error": str(error)})
            return
        except (UnicodeDecodeError, json.JSONDecodeError):
            self._send_json(400, {"error": "請求內容不是有效的 UTF-8 JSON。"})
            return
        self._send_json(200, {"ok": True})

    def _send_static_page(self, filename="night-sparrow-ordering.html"):
        page = ROOT / filename
        try:
            body = page.read_bytes()
        except OSError as error:
            print(f"Could not read the chat page: {error}", flush=True)
            self._send_json(500, {"error": "目前無法讀取問答頁。"})
            return
        self.send_response(200)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.send_header("X-Content-Type-Options", "nosniff")
        self._send_cors_headers()
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, format_string, *args):
        print(f"{self.address_string()} - {format_string % args}", flush=True)


def main():
    host = os.environ.get("HOST", "127.0.0.1")
    port_value = os.environ.get("PORT", "8000")
    try:
        port = int(port_value)
        server = ThreadingHTTPServer((host, port), LineBridgeHandler)
    except (ValueError, OSError) as error:
        raise SystemExit(f"無法在 {host}:{port_value} 啟動 LINE bridge：{error}") from error
    print(f"LINE bridge running at http://{host}:{port}/", flush=True)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        print("\nStopping LINE bridge.", flush=True)
    finally:
        server.server_close()


if __name__ == "__main__":
