---
layout: post
title:  "Automating super boring daily attendance checking on my corporate job"
date:   2018-04-20 17:19:27 +0700
---

> This article was previously published on [medium](https://medium.com/@rahmatnazali95/otomatiskan-hal-membosankan-dengan-python-pemindai-absensi-otomatis-584d2241f1){:target="_blank"}.

So I'm a freshgrad now and I landed a paid internship on a governmental-corporate job that has super formal culture.

The first thing I noticed something different is that I'm basically being paid daily based on my attendance.
The payday would still be on the early day of the month, 
but how much I'm being paid is not calculated by how much my performance on a certain job, 
but rather how good my attendance report was. This is the infamous advantage of civil servant (_ASN_) here on the nation where I grew up.

You are expected to log your attendance at 07.30 AM and log back on 17.00 PM before leaving.
Each subsequent 30 minutes you are late (or leaving early) will cut the total of your daily portion of your salary by a maximum of 50% (at least as far as I understand how this policy works).

Now I'm generally fine with such policy, as it prevents employee from being late or leaving too early.
But it doesn't really evaluate each employee's value to the organization.

Now I can log properly from the early morning and just spent my time drinking coffee or doing a subpar work, and the corp will still give me a full salary.
Couple of people around me probably working more than their limit but being late for 5 minutes due to the heavy traffic in Jakarta and their salary is being cut by some particular amount.
That's not fair!

Oftentimes, I'm wondering whether I really took this morning's attendance properly in the middle of working, and it really irritates me.
If I _did_ forget my morning attendance, I need to get it right quickly as fast as I can, and it happened a lot due to me being a fresh graduate with no previous working habits.

The corp provide me with an intranet site that shows my attendance report after being authenticated. This is more than fine and answers my problems.
But repeat that effort to open the site every day, and it adds up to a super boring task that I need to do several times a day.

| ![](/assets/2018-04-20/01_login_page.png) |
|:-----------------------------------------:|
|       The magnificent landing page        |

|                                                       ![](/assets/2018-04-20/02_report_page.png)                                                        |
|:-------------------------------------------------------------------------------------------------------------------------------------------------------:|
| Needs to manually filter all those number by eyes and allocate several seconds to conclude whether I missed my attendance or not (me being super lazy). |

Looking at the report page above, the site's dev did nothing wrong. The site does report it.
But the mere several seconds that I need to allocate from "where is today's report" to "whether I forgot my attendance" for everytime I need this to be done get super super boring. 
Not to mention opening my browser and the site, logging in, clicking ,etc.

Coming from a CS background, let's automate this repetitive task inside my CLI!

First, let's start by getting closer with the problem.

## Modelling the problem

There are 3 basic steps for a cycle of attendance checking:
1. Log in
2. Get monthly attendance report
3. Log out

Using Chrome's Developer Tools on each of the process above, we got several logs.

<details>
    <summary><strong>Long</strong> log images:</summary>
    <img src="/assets/2018-04-20/03_login_network.png" alt="login_network"/>
    <img src="/assets/2018-04-20/04_get_report_network.png" alt="get_report_network"/>
    <img src="/assets/2018-04-20/05_logout_network.png" alt="logout_network"/>
</details>
<br>

From those logs above, we can easily model each endpoint:

```
1. Log in
URL: /index.php/login
Status code: 302
Payload:
  - username
  - password
Returns:
  - PHPSESSID: a string token to be used as a Cookie

2. Get monthly attendance report
URL: /index.php/absensi/lihat_absensi_user
Status code: 200
Request Headers:
  - Cookie: <PHPSESSID that we got before>
Returns:
  - A full HTML page of montly attendance report

3. Log out
URL: /index.php/logout
Status code: 302
Request Headers:
  - Cookie: <PHPSESSID that we got before>
```

As per this state, we can easily automate these 3 task on some API tester like Postman.
But the attendance report endpoint returns a full HTML page. Something like this:

```html
<div id="dash_box_titlebar">LIHAT ABSENSI</div>
<div id="dash_box">
    <table border="0" class="nice-table">
        <tr bgcolor="EAEAEA" align="center">
            <td colspan="5">NAMA : RAHMAT NAZALI SALIMI</td>
            <td colspan="4">Nomor pegawai : * sensor :3 *</td>
        </tr>
        <tr align="center" bgcolor="EAEAEA">
            <td>No.</td>
            <td>Hari</td>
            <td>Tanggal</td>
            <td>Jadwal Masuk</td>
            <td>Masuk</td>
            <td>Jadwal Pulang</td>
            <td>Pulang</td>
            <td>Keterangan</td>
            <td>Lama TL/PSW</td>
        </tr>
        <tr bgcolor=#0084b2 class=simplebuttondua>
            <td align=center>1.</td>
            <td>Minggu</td>
            <td align=center>1-04-2018</td>
            <td align=center>00:00</td>
            <td align=center>00:00</td>
            <td align=center>00:00</td>
            <td align=center>00:00</td>
            <td align=center>Libur</td>
            <td align=center>-</td>
        </tr>
        <tr bgcolor=#EAEAEA class=simplebuttondua>
            <td align=center>2.</td>
            <td>Senin</td>
            <td align=center>2-04-2018</td>
            <td align=center>07:30</td>
            <td align=center>06:52</td>
            <td align=center>17:00</td>
            <td align=center>17:16</td>
            <td align=center>-</td>
            <td align=center>-</td>
        </tr>
        
        <!-- .... and so on until end of the month ... -->

    </table>
</div>
```

I don't think Postman alone can handle more sophisticated logic like get current day and parse all those HTMLs.
So we need another scripting language for this. Of course, I picked python!

There are nothing left to be drilled, so let's get to code!

## Coding

We are using python3 with these dependencies:
- [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/){:target="_blank"} for parsing HTML
- [request](https://requests.readthedocs.io/en/latest/){:target="_blank"} for doing HTTP

I'm also proudly using PyCharm, as I still have some weeks left of my educational license 🙂


Let's start by pinning all the variables we have before into reusable format: a `config.py`:

```python3
login_payload = {
    'url': 'http://10.10.10.10/index.php/login',
    'method': 'POST',
    'header': {
        'Cookie': None
    },
    'body': {
        'username': 'my username',
        'password': 'my password'
    }
}

get_monthly_attendance_report_payload = {
    'url': 'http://10.10.10.10/index.php/absensi/lihat_absensi_user',
    'method': 'GET',
    'header': {
        'Cookie': 'PHPSESSID that I got from login'
    },
    'body': {}
}

logout_payload = {
    'url': 'http://10.10.10.10/index.php/logout',
    'method': 'GET',
    'header': {
        'Cookie': 'PHPSESSID that I got from login'
    },
    'body': {}
}
```

Write the core parser, `parser.py`:

```python
import Config
import datetime
from bs4 import BeautifulSoup


class MonthlyData():
    def __init__(self, raw_monthly_page):
        self.soup_instance = BeautifulSoup(raw_monthly_page, 'html.parser')
        self.monthly_record = self.soup_instance.find_all('tr', class_ = 'simplebuttondua')
        self.current_date = datetime.datetime.today().strftime('%d-%m-%Y')
        self.current_date = self.current_date.replace('0', '',  1) if self.current_date.startswith('0') else self.current_date
        self.current_time = datetime.datetime.today().strftime('%H:%M')

    def get_daily_report(self):
        for daily_report in self.monthly_record:
            if daily_report.contents[3].text == self.current_date:
                nomor = daily_report.contents[1].text
                hari = daily_report.contents[2].text
                tanggal = daily_report.contents[3].text
                morning_attendance = daily_report.contents[5].text
                leaving_attendance = daily_report.contents[7].text

                self.morning_attendance_limit = daily_report.contents[4].text
                self.leaving_attendance_limit = daily_report.contents[6].text

                # pretty prints it
                print('\n')
                print('\t' + hari, tanggal, ' |', self.morning_attendance_limit, '->', self.leaving_attendance_limit)
                print()
                print('\tWaktu saat ini    :', self.current_time)
                self.print_today_report('\tWaktu Absen Masuk :', morning_attendance)
                self.print_today_report('\tWaktu Absen Pulang:', leaving_attendance, isPulang = True)
                print('\n')

    def print_today_report(self, header, attendance_time, is_leaving = False):
        if attendance_time == '00:00' or (is_leaving and self.current_time <= self.leaving_attendance_limit):
            message = '\t  Bukan/belum periode absen'
        elif attendance_time == '':
            message = '\tX Belum Absen'
        else:
            message = '\tV Sudah absen'
            
        print(header, attendance_time if attendance_time != '' else '-----', message)
```


Now let's create the main scrapper, `scrapper.py`:

```python
import Config
import requests

class Scrapper():
    def __init__(self):
        self.cookies = None
        self.attendance_data = None

    def login(self):
        result = requests.post(Config.loginPayload['url'], Config.loginPayload['body'])
        assert result.status_code == 302
        self.cookies = result.cookies

    def get_attendance_report(self):
        assert self.cookies is not None
        result = requests.get(Config.getAbsenPayload['url'], Config.getAbsenPayload['body'], cookies = self.cookies)
        self.attendance_data = result.text

    def get_today_attendance(self):
        assert self.attendance_data is not None
        monthly_data = MonthlyData(self.attendance_data)
        monthly_data.get_daily_report()

    def logout(self):
        assert self.cookies is not None
        result = requests.get(Config.logoutPayload['url'], Config.logoutPayload['body'], cookies = self.cookies)

    def scrap(self):
        self.login()
        self.get_attendance_report()
        self.get_today_attendance()
        self.logout()
```

and finally calls it from `main.py`:

```python
import Core

if __name__ == '__main__':

    scrapper = Core.Scrapper()
    scrapper.scrap()

    print('\n\n\tTekan sembarang untuk keluar')
    input()
```


## Testing the code

I won't deliberately come late just to test this 😊
So I mocked the code to make it look like I weren't taken a proper attendance on a particular morning, and this is what I got:

|                      ![](/assets/2018-04-20/06_test_1.png)                       |
|:--------------------------------------------------------------------------------:|
| A red text indicating that my morning attendance is not logged on the system yet |

And when I disabled the mock (so the program could read my today's morning attendance), I got this:

|   ![](/assets/2018-04-20/07_test_2.png)    |
|:------------------------------------------:|
| A beautiful green as in "it has been seen" |

It seems to be working!


## Finishing

To put a cherry on top of the cake, I put a shortcut to my desktop when I need urgent checking:

| ![](/assets/2018-04-20/08_polish_1.png) |
|:---------------------------------------:|
|     Did you see my favourite game?      |

And run it form the desktop:

|       ![](/assets/2018-04-20/09_polish_2.png)       |
|:---------------------------------------------------:|
| Just realize `cmd` doesn't seem to support color 😔 |

And that's it! We just automated all of this boring nonsense to a simple two clicks: opening the program and closing it.

If I could go more with this, I could add a 10 seconds timer to auto close the program so I just need 1 click now 👹

Now let me drink some hot tea while people around me spend several minutes doing this everyday for the rest of their life.
