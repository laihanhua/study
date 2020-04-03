## go build

1. 可能是gunicorn workertimeout了，此时才报那个msg_json错误
2. 自己测试raise 一个error，gunicorn不会worktimeout,只会打印错误日志，此时返回internal server error（/Library/Python/2.7/site-packages/web/webapi.py）
3. 
