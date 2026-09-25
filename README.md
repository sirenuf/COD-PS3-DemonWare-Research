# Summary
Black Ops, Modern Warfare 2 and Modern Warfare 3 all have issues on the PS3. These issues are caused by an underlying missmatch around the so called XUID new accounts and old accounts who have had their PSN username changed face, which came in effect around 2018. The fix is to use the XUID retrieved from the Call of Duty servers, and not the locally computed one.

Please consult the **glossary** at the bottom of the page for acronyms and other definitions. Like TU, XUID etc.

# Introduction
Several Call of Duty titles released for the Playstation 3 are plagued by a variety of issues for new accounts, and to my knowledge the root cause of these issues have never been properly discovered, documented or talked about until now.

Here I am going to share what the cause is and how to fix it.

# What are wrong with these games?
To my knowledge as of writing this, the **following games have issues** today for **new accounts** that haven't played a Call of Duty **prior to somewhere around 2018**.
* **Modern Warfare 2 (IW4)** - Progression/stats do not save.
* **Black Ops 1 (T5)**		 - Progression/stats do not save.
* **Modern Warfare 3 (IW5)** - A pop up containing *"Communication with the Activision servers has been interrupted"* displays when trying to join a server, public or private. Boots you back to the main menu and crashes others' servers.
* **Black Ops 2 (T6)** 	 	 - Freezes if you're logged in on PSN when starting the game.
* Black Ops 3 (T7) 			 - Crashes when you unlock a new item.

*The scope for this document will only contain the first four titles, ignoring the last title "Black Ops 3", as I do not have interest in this title for this platform.*


# Previous Community Workarounds
To aid these problems, the community has provided workarounds and I will go over them here, and argument for why they aren't sustainable. <br>
*It is also worth noting that a lot of these workarounds were developed and primarily targetted by and for script kiddies and modders who had their main account banned for cheating or something else, and needed a solution in order to play on a new throwaway account to continue cheating.*

## Modern Warfare 2 - live RTM
To my knowledge, the only solution found for Modern Warfare 2 was peeking into live memory locations inside the game for where stats actually do get stored, saving the live memory to a file statically and updating it periodically, either automatically or manually via a mod menu or similar program that injects into the game. <br>
Others resorted to just sending unlock all payloads to these memory locations on startup, also most commonly found on mod menus. <br>
Sometimes, this didn't cover custom classes.

This is referred to as **RTM**, or **Real Time Modding**.

## Black Ops 2 - Stay signed out and wait
Before launching BO2, make sure you're signed out of PSN. It will then successfully put you into the multiplayer menu without freezing. After reaching the menu however, wait for roughly 40 seconds. Seriously. After waiting, you can sign into PSN and play the game like normal with no crashing.

## Black Ops 1 - live RTM
Black Ops 1 share the same community workarounds as Modern Warfare 2. It is worth noting though that Black Ops 1 is the trickiest of them all as every different title ID/game ID I've seen used all have slightly shifting memory values. I haven't checked disassemblies/diff to see what actually causes this but segments in memory are usually about 128 - 192 bytes off from each other depending on title ID.

## Modern Warfare 3 - XUID spoofing and detour
### Detouring
While pretty gatekept, the actual code that both kicks you out as a new account, and that makes your server crash as a host when a new account attempts to join is the result of just one function. Specifically, `dwFetchPerformanceValuesComplete`.

For the record, this is the decompilation with IDA's PPC decompiler from a retail Xbox 360 debug build of MW3:

```c
taskCompleteResults __fastcall dwFetchPerformanceValuesComplete(
        overlappedTask *const task,
        PlayerRank *playerRanks,
        int *numPlayerRanks)
{
  bdRemoteTask::bdStatus v6; // r25
  bdLobbyErrorCode ErrorCode; // r26
  int v8; // r27
  int WindowCredit; // r31
  unsigned __int64 v10; // r9
  int v11; // ctr
  PlayerRank *v12; // r10
  unsigned __int64 *p_m_entityID; // r11
  char v15; // [sp+50h] [-80h] BYREF

  if ( !task || !task->u.remoteTask.m_ptr || !dwGetMatchmaking(task->controllerIndex) )
    return TASK_ERROR;
  dwLobbyPump(task->controllerIndex);
  v6 = task->u.remoteTask.m_ptr->getStatus(task->u.remoteTask.m_ptr);
  ErrorCode = bdRemoteTask::getErrorCode(task->u.remoteTask.m_ptr);
  if ( v6 == 2 )
  {
    v8 = *numPlayerRanks;
    WindowCredit = bdSAckChunk::getWindowCredit((bdSAckChunk *)task->u.remoteTask.m_ptr);
    memset(playerRanks, 0, 16 * v8);
    HIDWORD(v10) = 0;
    *numPlayerRanks = 0;
    if ( WindowCredit > 0 )
    {
      v11 = WindowCredit;
      v12 = playerRanks;
      p_m_entityID = &s_performanceValue[0].m_entityID;
      do
      {
        if ( SHIDWORD(v10) < v8 )
        {
          v10 = p_m_entityID[1];
          v12->rank = v10;
          v12->xuid = *p_m_entityID;
          ++*numPlayerRanks;
        }
        ++HIDWORD(v10);
        p_m_entityID += 3;
        ++v12;
        --v11;
      }
      while ( v11 );
    }
  }
  if ( ErrorCode )
  {
    TaskManager_ClearTask(task);
    if ( ErrorCode == BD_TOO_MANY_TASKS )
      Live_ThrowError(ERR_DROP, "XBOXLIVE_TOOMANYTASKS");
    dwLobbyErrorCodeToString(ErrorCode, &v15, 0x40u);
    Live_ShutdownDueToTerminalError(task->controllerIndex);
    Live_ThrowError(ERR_DROP, "XBOXLIVE_LIVEERROR");
  }
  return dwTaskStatusConvert(v6, ErrorCode);
}
```
Like the **XUID**, it seems like when designing the IW engine at Infinity Ward, they targetted the Xbox as their main branch of development, as despite the engine also targetting PC and the PS3, functions still share the same **XBOX adjacent** nomenclature.

The location of the function entry in the last TU 1.24 on the PS3 is vaddr `0x340C8C`. Decompiling that will give you basically a 1:1 result of the 360 retail variant, just no symbols.

So mod menus use this to their advantage and just detour this function to do nothing, `return 0`. And that is the most straight forward band aid fix and at the same time necessary fix, as having it in your game will also result in people being able to crash your server as host at will by just joining your server via a direct invite with a new account.

Read more about it [here: **setsid - MW3 PSN Fix**](https://github.com/setsid/mw3-ps3-psn-fix)<br>
*Slightly different approach but carries the same result.*

### XUID Spoofing
The other method was by spoofing your so called **XUID**, which will become much more relevant later in this document.
The XUID is a 64 bit (8 byte) identifier used across all platforms for multiplayer accounts on Call of Duty.<br>

The name XUID is derived from Xbox account API's actual identifier - XUID =  **X**box **U**ser **ID**entifier. <br>
It has absolutely nothing to do with Xbox APIs in the PC and PS3 builds of these games, it just shares the name, incredible I know.

To get someone's XUID, you'd simply use the `Tiger192` hashing function of your current PSN **online ID** which directly corresponds to your current PSN username, truncate it to the first 64 bits, and reverse the byte order. I have a bash helper function called **xuid.sh** at my [GitHub gist](https://gist.github.com/sirenuf/66d13d62f72eaea986d67fb446bace88) which does just that:

For example, the XUID of my PSN account would be <br>
```bash
$ ./xuid.sh viktor6153
0x5E00DAC667285AC0
```

Where **viktor6153** is my current PSN username (online ID). <br>
So my XUID of my PSN account is `5E 00 DA C6 67 28 5A C0`.

If you scan for your own XUID in memory on for example MW3 while logged into PSN; you can then replace those values with someone else's XUID and trigger a profile refresh by changing the value at vaddr `0x1BBBC29` to `u8 0x01`.

That's what mod menu's did. They spoofed your current XUID to a XUID belonging to an old account. This change in identity from a new account's XUID to an old account's XUID fixed all issues encountered on new accounts, so it is safe to assume that **something happened** in 2018 which led to new accounts' XUIDs failing to get validated, which may be the cause for the other games. More on this later.

### Why not just resort to spoofing to someone else's XUID if it works?
Well, it works until it doesn't. If someone in the same lobby as you has the same XUID as someone else, then the engine will unironically start tweaking. Both players sharing the same XUID will have their names shifting in the lobby until one or both players gets kicked out. And there is no known way in advance to tell in advance before connecting if someone already has the same XUID as you. Nor is it possible to change your XUID identity while in a game.

It is safe to say that this isn't a reliable method by any means. Mod menus seemed to circumvent this by having an array of 200 or more XUIDs to choose from when starting the game, in hopes of never running into someone with the same XUID, and never diagnosing what the underlying issue actually is.

### Why not just resort to detouring `dwFetchPerformanceValuesComplete`?
If the host of a server runs an untouched version of MW3; that being a version that don't have `dwFetchPerformanceValuesComplete` patched themselves, then everytime you try to connect as a new account, their server will crash like usual.

So to make this workaround work properly, every single host has to have the patch. As soon as a host doesn't have the patch, the server goes down.

# What now?
For a while I didn't really know. That was until the awesome developer and reverse engineer [Jacob Schroeder](https://github.com/jacob-schroeder) started his work on his collection of [patched MW2 builds](https://github.com/jacob-schroeder/IW4-Binaries).

To his findings, the IW engine has technically two means of retrieving your XUID. When you first sign in to COD with PSN, you make an **auth request** to Demonware. <br>
Demonware is the middleware that Activision uses for its Call of Duty games since 2005, it handles account authentication, account stats storage, tracking, and matchmaking. If not more.

The response from the aforementioned **auth request** contains your XUID that is stored on the Demonware servers. For old accounts, this XUID checks out to be derived from your online ID, like normal. However, for new accounts, this XUID turned out to be entirely different.<br>
For my personal account, viktor6153, which is in fact a "new" account that has never played any COD game on the PS3 before, the XUID from Demonware was not `5E 00 DA C6 67 28 5A C0`. Instead, it was `73 92 34 CE C1 95 7D 40`.

The other way of retrieving an XUID is a local function in the engine, which initially computes your XUID via the traditional way of doing the Tiger hash on your online ID and putting it in a cache, so the engine can quickly retrieve your XUID. This is what is causing the actual issues for new accounts today. The engine tries to match `5E 00 DA C6 67 28 5A C0` with `73 92 34 CE C1 95 7D 40`, which are two completely different identifiers, and fails. The fix that Jacob did for MW2 was as easy as just putting the Demonware XUID response into the XUID cache, instead of getting it locally. And ultimately, for MW2, this solved the issue.

### Jacob's original helper:
```asm
006EC100 lwz r0,0x98(r1)      ; replay displaced instruction
006EC104 ld r8,0xB0(r1)       ; bdAuthTicket.user_id
006EC108 cmpdi r8,0
006EC10C beq 0x006EC128       ; zero keeps stock fallback
006EC110 lis r11,0x73
006EC114 lwz r11,-0x4534(r11) ; *(u32 *)0x0072BACC
006EC118 std r8,8(r11)        ; cache server user ID
006EC11C lwsync
006EC120 li r10,1
006EC124 stb r10,4(r11)       ; publish valid last
006EC128 b 0x004033F4
```

## Mystery has finally been solved
I could very quickly verify that the root cause of MW3 and BO1 is the exact same as MW2. By patching the local XUID function to just always return `73 92 34 CE C1 95 7D 40` - my XUID retrieved from Demonware - stats started saving on BO1 and I could play games just fine on MW3 without patching any function or spoofing my XUID. If I searched for `73 92 34 CE C1 95 7D 40` in memory in both BO1 and MW3, I could also verify that they are sitting there, which confirms that MW3 and BO1 also retrieves this from Demonware.

Now because of just the natural progression of development, underlying engine modifications, changes in the Demonware SDK, the previous solution of just retrieving the XUID from the Demonware auth response and putting it in the local cache wasn't as straightforward in both BO1 and MW3. With the help of Claude and a bit of reverse engineering however, I could very easily port this to [my project **CODPatch-PS3**](https://github.com/sirenuf/CODPatch-PS3) to make this fix portable (albeit with a wildcard memory signature scanner for BO1 because of the aforementioned jumping offsets, and an additional thread which watches and corrects the identity post auth. You don't notice any of this however).

## What about BO2?
I am 99.999% sure BO2 suffers from the exact same underlying XUID missmatch. The reason why I won't go over it however is because I don't need to. [setsid's **BO2 Freeze Patch**](https://github.com/setsid/bo2-ps3-psn-freeze-fix) addresses the freezing issue, and the fix is legit to NOP one line in all three different executables. Reading his report, I think it is safe to assume that the issue is also related to the our broken 64 bit XUID number.

After NOPing these lines, with no XUID adjustment, the game plays and runs exactly how it should. So there is no real reason to keep trying to find out on a deeper level at how the T6 engine works. If you're capable and interested though, issues and PRs are welcome.

# Why does the XUID even differ?
Well to start of, in 2019 Sony decided to introduce the ability to change your current PSN username (aka online ID). It was in 2018 when Sony started playing with this idea, [adding it to their APIs](https://kotaku.com/game-developers-say-theyre-preparing-for-psn-name-chang-1829521254).

Now this is only one source, but reading a lot of these reports and going back [further to 2016](https://fenixbazaar.com/2016/11/17/psn-id-change/), all of them mention the phasing out of the online ID as being your primary identifier, and [instead using your account ID](https://www.neogaf.com/threads/ps4-sdk-updates-hint-at-possibly-allowing-psn-id-changes-in-the-future.1314948/). Those release notes for Unreal Engine 4 are very interesting, it was per Sony's request to start using account IDs instead of online IDs, as early as 2016.

Since there is no realistic way of reversing a truncated result of a hashing function to find out what the input could be, you had to resort to this kind of bruteforce trial and error style to see what is what. After all of this research I did however to see what actually changed around this time, I decided to try to input my account ID to my bash function to see if it is what is actually passed through now. <br>
You can find your account ID as the first 8 bytes in the `np_cache.dat` binary file inside your home directory on your PS3. For example, my path was `/dev_hdd0/home/00000013/np_cache.dat`. You can also just very easily look this up online via public APIs such as [PSNAWP](https://github.com/isFakeAccount/psnawp) or a third party PSN account query website.

As it turned out, my account ID for viktor6153 is `8534295246308455420`. And lo and behold, entering it in my helper:
```bash
$ ./xuid.sh 8534295246308455420
0x739234CEC1957D40
```

Yup, that checks out. In case you don't remember, my actual XUID for viktor6153 as Demonware puts it is `73 92 34 CE C1 95 7D 40`, which is what we just successfully hashed via the account ID. <br>
This means that we can now guarantee that the XUID for new accounts is going to be derived from the account ID instead, not the online ID. That is the issue we have been dealing with the entire time. <br>
Demonware must've changed their APIs to generate new XUIDs for their new games to use the new Sony toolkit's Account ID recommendation, which consequently broke these old unsupported titles, which were programmed to always assume that XUIDs are derived from the user's current Online ID.

*Remember to always use static unique user identifiers for authentication kids.*

To fix these issues, check out [See Also](/#See_also) or specifically [my plugin](https://github.com/sirenuf/CODPatch-PS3).

# See also
* [setsid - **PS3 Tools**](https://github.com/setsid/ps3-tools)
* [setsid - **BO2 PSN Freeze Fix**](https://github.com/setsid/bo2-ps3-psn-freeze-fix)
* [setsid - **MW3 PSN Fix**](https://github.com/setsid/mw3-ps3-psn-fix)
* [Jacob Schroeder - **IW4 Studio**](https://github.com/jacob-schroeder/IW4Studio)
* [Jacob Schroeder - **IW4 Binaries**](https://github.com/jacob-schroeder/IW4-Binaries)
* [sirenuf (me) - **CODPatch-PS3**](https://github.com/sirenuf/CODPatch-PS3)
# Credits
* [**setsid**](https://github.com/setsid) - Amazing work and research done to make Call of Duty on the PS3 a better place for everyone.
* [**Jacob Schroeder**](https://github.com/jacob-schroeder) - For his countless patches and original discovery and solution for MW2. If it weren't for him, none of this would've been possible.
* [**Yausent**](https://github.com/yausent) - Countless resources around this dead scene, and always being down to help me out in testing. 

# Glossary
* **XUID** - A unique 64 bit identifier. Originally stands for Xbox User Identifier. Don't be fooled by the name, it has absolutely nothing to do with Xbox on the PS3. It is the identifier the game engine uses for multiplayer accounts on COD on the PS3.
* **Online ID** - Your current PSN username.
* **Account ID** - A unique 64 bit identifier PSN accounts have, supplied by Sony.
* **T5** - The codename for Black Ops 1. Also the unofficial name for the BO1 game engine.
* **T6** - The codename for Black Ops 2. Also the unofficial name for the BO2 game engine.
* **IW4** - The codename for Modern Warfare 2. Also the unofficial name for the MW2 game engine.
* **IW5** - The codename for Modern Warfare 3. Also the unofficial name for the MW3 game engine.
