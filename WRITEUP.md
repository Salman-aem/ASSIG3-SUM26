# Assignment 3 Writeup

## FIX 1 — SQL Injection Login Bypass

The vulnerable part was in the `/login` POST route in `server.js`. The problem was that the app was putting the username and password directly inside the SQL query. So if someone types SQL code in the username box, the database can read it like part of the command, not just normal text.

The payload I used was:

curator' --

This payload closes the username part and comments out the password check. Because of that, the app could log in as `curator` without the real password.

The fix was to use a parameterized query with `?` placeholders. This keeps the username and password separate from the SQL command.

This works because the database treats the input as data only, not as SQL code.

Limit: this fixes the login query, but other SQL queries in the app also need to be fixed the same way.


## FIX 2 — SQL Injection UNION Data Dump

The vulnerable part was in the `/search` route in `server.js`. The problem was that the app was putting the search term and sort value directly inside the SQL query. Because of that, an attacker could use UNION injection to pull data from another table.

The payload I used was:

%' UNION SELECT 999, group_concat(username || ':' || password, ' | '), 'dump', 'dump' FROM users-- 

This payload made the search result return usernames and passwords from the users table.

The fix was to use `?` placeholders for the search term and an allow-list for the sort option. The search value is now treated like normal text, and the sort value can only be one of the safe options.

This works because the user cannot add extra SQL commands through the search box or the sort field anymore.

Limit: this fixes the search route, but any other SQL query in the app also needs to use safe parameterized queries.


## FIX 3 — Reflected XSS

The vulnerable part was in the search page. The app was showing the search value back on the page without encoding it first. So if someone searched using HTML or JavaScript, the browser could treat it like real code.

The payload I used was:

<img src=x onerror=alert(1)>

This payload was reflected back into the page as live HTML.

The fix was to use `escapeHtml(q)` before showing the search term on the page.

This works because characters like `<`, `>`, and quotes get changed into safe text before the browser reads them.

Limit: this only protects this search output. If the app prints user input somewhere else, that place also needs output encoding.


## FIX 4 — Stored XSS

The vulnerable part was in the comments section. The app stored comments and then showed them back to every visitor without encoding the comment text. This is dangerous because the bad comment stays saved in the app.

The payload I used was:

<img src=x onerror=alert(1)>xss_test

This payload was stored as a comment, and when the listing page loaded, it came back as live HTML.

The fix was to encode the comment output using:

escapeHtml(c.body)
escapeHtml(c.author)
escapeHtml(c.created_at)

This works because the stored comment is shown as text instead of being treated as HTML code.

Limit: this protects the comments section, but if comments are displayed somewhere else later, that area also needs encoding.


## FIX 5 — Cookie Flags and CSP

The vulnerable part was the session cookie and the missing CSP header. The cookie did not have enough protection, and the app did not send a Content Security Policy header.

The fix was to add these cookie flags:

HttpOnly; SameSite=Lax

I also added this Content Security Policy:

default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; object-src 'none'; base-uri 'self'; frame-ancestors 'none'

This works because `HttpOnly` helps stop JavaScript from reading the cookie, `SameSite=Lax` helps reduce cross-site request risk, and CSP limits what scripts and resources can run on the page.

Limit: cookie flags and CSP help reduce the damage, but they do not replace fixing SQL injection and XSS directly.


## Final Test Result

After all fixes, I ran:

npm run exploit

The result showed:

[1] SECURE
[2] SECURE
[3] SECURE
[4] SECURE
[5] SECURE

RESULT: all five defects are closed.