# UIUCTF 2021 - CEO Writeup

CEO was a "misc" challenge during UIUCTF 2021 in which you need to inspect and gather creds from a `.pcap` file. It was rated as a "beginner" challenge in a CTF that was _extremely_ challenging for myself, and forced me to dust off some serious CTF cob webs. While it's not a very glamorous chall, this writeup aims to simply note some of the techniques I used to get the flag. So with that said, let's jump in.

## The Prompt
Viewing the challenge, we're prompted with the following note.

![chall](https://user-images.githubusercontent.com/51839088/129170333-68eb291e-509c-4255-847e-181e1753b41c.png)

**Note: During the initial stages of the CTF, this sentence was not in the challenge prompt: "Wrap the password in the flag format." 
This led me down some rabbit holes that we'll see later.**

## Initial Analysis
After downloading the `.pcap` and opening it in Wireshark, we can see there's quite a bit of traffic, mostly using the 802.11 protocol.

![Wireshark](https://user-images.githubusercontent.com/51839088/129171054-24fb4e55-178b-4eab-8302-c9f5baa95c56.png)

If we click `Wireless > WLAN Traffic` in Wireshark we can see that the SSID related to this captured traffic is: `joesheer`.

![SSID](https://user-images.githubusercontent.com/51839088/129171260-db6ac97e-1126-4425-b357-a023d4b2f7bb.png)

## Original Rabbit Hole
So as I mentioned earlier, the original challenge prompt was a little.... well a whole lot different. 
If you clicked the original hint it mentioned doing some OSINT on the CEO to get the Wi-fi password.

I did some research and found that if you know the password used in captured 802.11 traffic, you can enter the password and wireshark will decrypt it.

We can do this pretty easily by right clicking a captured 802.11 packet and hitting:

`Protocol Preferences -> IEEE 802.11 Wireless LAN -> Decryption Keys...`

Then create a new entry with the little plus at the bottom, select `wpa-pwd` as the key type, and enter the password in the `key` field.

![decryption keys](https://user-images.githubusercontent.com/51839088/129172075-040d5d61-d5b6-4206-bf98-18549d5153e0.png)

Anyways, knowing this my train of thought for the challenge was:
1. Get potential password from OSINT on the CEO
2. Enter password in Wireshark
3. Find key in decrypted traffic

Using the information we've gathered so far, we can find the CEO's twitter account pretty easily:

![Google searching](https://user-images.githubusercontent.com/51839088/129172523-279c1a3d-f473-437f-a9a6-5153e44dda64.png)

![CEO twitter](https://user-images.githubusercontent.com/51839088/129172556-72534735-4e1c-4773-aa0d-562654f833ec.png)

And this is where the rabbit hole begins. 
I spent more time than I'd like to admit searching through this account, his followers, who he's following... their tweets... all to no avail.

At a certain point I got desparate and wondered if I should just start guessing based on what he was talking about. After all it's a beginner challenge right?

Here's my original wordlist concocted from his tweets/rewteets:
```bash
~/Downloads/wifi:> $ cat fuckme.txt         
megacorpone
megacorp
nanotech
nanotechnology
nano-technology
MegaCorp_One
MegaCorp
DavidBeckham
USA
US
equestrian
horse
horses
soccer
football
manchesterunited
manu
manunited
```

I then manually entered each one into the Wireshark decryption keys section. While this didn't work, as you'll see later I actually wasn't far off.

## Flag
Some frustrated research later led me to realizing there's an alternate path to success.

Wi-fi traffic typicall contains a 4-way handshake used to encapsulate the packets between a router and client.

We can easily view this handshake by using the `eapol` filter in Wireshark.

![viewing eapol](https://user-images.githubusercontent.com/51839088/129173831-6d979fa8-e85d-4ab8-989e-338d0e8a549d.png)

This essentially just means that the client and host have agreed to talk to eachother, and the traffic is encapsulated.

What this also means for us is that there's a valid WPA key (we already knew this but pretend you didn't) that we can crack.

### Cracking the password
To crack the captured handshake with Hashcat, we'll need to use `cap2hccapx` to get it into the [hccapx format](https://hashcat.net/wiki/doku.php?id=hccapx).

This most likely won't be in your path by default (if you're using Kali), but we can locate it pretty easily.

![cap2hccapx](https://user-images.githubusercontent.com/51839088/129174463-3c3555c9-1ab5-4275-bee9-9b1a757bef83.png)

After that we just give it the `.cap` file and a name to write to:

![Getting a hccapx](https://user-images.githubusercontent.com/51839088/129174610-be5c4e09-93c3-48aa-9e20-07cfc30696b7.png)

Now we can use Hashcat to crack it.

`~/ctf/uiuctf_2021/misc/CEO:> $ hashcat -m 2500 hccapx-out /usr/share/wordlists/rockyou.txt --force `

![](https://user-images.githubusercontent.com/51839088/129174858-af1113b3-7f0f-4288-bd57-3503f09eff5b.png)

Now we can submit the flag! Except... well I didn't initially.

So I originally didn't realize that the Wi-Fi password was the actual flag. 

I falsely assumed that you must use the password to decrypt the traffic then find the flag in some decrypted packet.

After aimlessly searching through packets, I noticed the challenge writeup was updated and submitted the flag I already had.

Done!

## Closing Thoughts

You might also notice that the cracked password was actually in my original manual wordlist from Twitter. I'll be honest I can't remember if I noticed that the traffic was decrypted with this password or not. This just added to the overall frustration of the challenge. Unfortunatel as is usually the case with annoying challs I learned allot. 

Thanks for reading :)

_Written by 0xAshh_

## Hit me up
- [Twitter](https://twitter.com/0xAshhh)
- [GitHub](https://github.com/0xash)
- Discord: Ashh#1446
