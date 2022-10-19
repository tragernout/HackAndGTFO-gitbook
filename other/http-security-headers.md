# HTTP Security Headers

**HTTP Security Headers** - это HTTP-заголовки для обеспечения безопасности веб-сайта.

### CORS

### X-Frame-Options

Может использоваться для того, чтобы показывать браузеру отображать ли страницу в тегах \<frame>, \<iframe>, \<embed> или \<object>. Сайты могут использовать этот заголовок, чтобы избежать атаки Click-jacking.

### Content-Security-Policy || X-Content-Security-Policy

По дефолту CSP ограничивает исполнение JavaScript путем запрета встроенных скриптов и динамической оценки кода.

Чтобы включить CSP, нужно настроить сервер так, чтобы он возвращал заголовок Content-Security-Policy или X-Content-Security-Policy (устаревший). Также можно использовать тег \<meta>:

```
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; img-src https://*; child-src 'none';">
```



### HSTS

### HTTPOnly Cookie

### X-Forwarded-For

