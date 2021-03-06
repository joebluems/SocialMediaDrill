# SocialMediaDrill
This is an Apache Drill do-it-yourself workshop that will help you explore and process social data, which has many business applications, including the following:
-	Determine the satisfaction of related users (i.e. customers) by monitoring sentiment and matching entities
-	Detect trends affecting your customers or your industry (i.e. high check fees, slow customer service, etc.)
-	Process and analyze large amounts of text with combinations of NLP and LDA (i.e. topic modeling)
-	Extract common affiliates to find similar users and expose graph capabilities (i.e. users who like similar posts, topics, etc. create a community, etc.)

<br>
## Get Apache Drill
Link for the download: http://drill.apache.org/download/ <br>
Follow instructions to setup (i.e. JAVA_HOME) and install Drill <br>
Launch drill from the main drill folder with this command: <b>./bin/drill-embedded</b> <br>
<br>
## Parsing Facebook Messages in Drill
One of the benefits of Apache Drill is reading nested data, such as the format of facebook messages, at scale, often bypassing lengthy loading and formatting processes. In this workshop, we will download some data and use Drill to explore the format to extract usable information.

To gain limited access to a Facebook feed, you must generate an Oauth token for an access key. One place you can to that is here (using your personal facebook account):
http://apigee.com/console

An example of an access code is the following:
'EAAKMrAl97iIBAPzIL4w…OtqWmD04MIZD'

Once you get the code and generate a URL for a specific Facebook account, you can grab some posts using this format:
```
curl ‘url_for_facebook_access’ > output_file.json
```
Sample feeds are included in the repository if you do not wish to generate your own files.  For this example, we will use the messages on the page of the Metropolitan Transportation Authority (i.e. NYC subway, buses, etc.). Their page can be located here: http://www.facebook.com/MTA.info/

Here are a few lines from the <b>subway.json</b> file that demonstrate the nesting in the output:
```
{
   "data": [
      {
         "id": "250313209090_10154079915599091",
         "from": {
            "name": "Metropolitan Transportation Authority (MTA)",
            "category": "Government Organization",
            "id": "250313209090"
         },
         "message": "Big news for MTA Bridges and Tunnels!",
         "message_tags": {
            "13": [
               {
                  "id": "244246159423",
                  "name": "MTA Bridges and Tunnels",
                  "type": "page",
                  "offset": 13,
                  "length": 23
               }
            ]
         },
         "link": "http://www.mta.info/news-governor-cuomo-bridges-and-tunnels-led-lights-open-road-tolling-automatic-tolling/2016/10/05",
         "name": "MTA | news | Governor Cuomo Announces Transformational Plan to Reimagine New York's Bridges and Tunnels for 21st Century",
         "caption": "mta.info",
         "description": "\"Open Road\" tolling and colorful LED lights are among the changes you'll see in the coming years when traveling on MTA's Bridges and Tunnels. Governor Andrew M. Cuomo today announced a transformational plan to reimagine New York's crossings for the 21st century. The plan will institute state-of-the-...",
         "icon": "https://www.facebook.com/images/icons/post.gif",
…
}
```
The json file has two columns: data and paging. Let’s look at one record. All of the queries in this document should work as copy-and-paste. You will need to change the location on my laptop to yours.

Be careful that your system is not converting the ticks (`) to a single quote (‘). <br>
Also, omit the drill prompt (i.e. <b>0: jdbc:drill:zk=local> </b>)  during copy. 
```
0: jdbc:drill:zk=local> select * from dfs.`/Users/joeblue/sentiment/subway.json`;
+------+--------+
| data | paging |
+------+--------+
| [{"id":"250313209090_10154079915599091","from":{"name":"Metropolitan Transportation Authority (MTA)","category":"Government Organization","id":"250313209090"},"message":"Big news for MTA Bridges and Tunnels!","message_tags":{"13":[{"id":"244246159423","name":"MTA Bridges and Tunnels","type":"page","offset":13,"length":23}],"4":[],"40":[],"12":[],"0":[],"55":[]},"link":"http://www.mta.info/news-governor-cuomo-bridges-and-tunnels-led-lights-open-road-tolling-automatic-tolling/2016/10/05","name":"MTA | news | Governor Cuomo Announces Transformational Plan to Reimagine New York's Bridges and Tunnels for 21st Century","caption":"mta.info","description":"\"Open Road\" tolling and colorful LED lights are among the changes you
…
+------+--------+
1 row selected (0.285 seconds)
```
We want the “data” column, which is where the nested information is located. <br>
Issue a query to look at the first message in the data list:
```
0: jdbc:drill:> select data[0] as message from dfs.`/Users/joeblue/sentiment/subway.json`;
+---------+
| message |
+---------+
| {"id":"250313209090_10154079915599091","from":{"name":"Metropolitan Transportation Authority (MTA)","category":"Government Organization","id":"250313209090"},"message":"Big news for MTA Bridges and Tunnels!","message_tags":{"13":[{"id":"244246159423","name":"MTA Bridges and Tunnels","type":"page","offset":13,"length":23}],"4":[],"40":[],"12":[],"0":[],"55":[]},"link":"http://www.mta.info/news-governor-cuomo-bridges-and-tunnels-led-lights-open-road-tolling-automatic-tolling/2016/10/05","name":"MTA | news | Governor Cuomo Announces Transformational Plan to Reimagine New York's Bridges and Tunnels for 21st Century","caption":"mta.info","description":"\"Open Road\" tolling and colorful LED lights are among the changes you'll see in the coming years when traveling on MTA's Bridges and Tunnels. Governor Andrew M. Cuomo today announced a transformational plan to reimagine New York's crossings for the 21st century. The plan will institute state-of-the-...","icon":"https://www.facebook.com/images/icons/post.gif"," …
+---------+
1 row selected (0.146 seconds)
```
Lots of nesting going on there. Let’s pull out some of the fields nested under data:
```
0: jdbc:drill:> select data[0].id `messageID`,data[0].`from` `sender`,data[0].message `message` from dfs.`/Users/joeblue/sentiment/subway.json`;
+---------------------------------+------------------------------------------------------------------------------------------------------------------+----------------------------------------+
|            messageID            |                                                      sender                                                      |                message                 |
+---------------------------------+------------------------------------------------------------------------------------------------------------------+----------------------------------------+
| 250313209090_10154079915599091  | {"name":"Metropolitan Transportation Authority (MTA)","category":"Government Organization","id":"250313209090"}  | Big news for MTA Bridges and Tunnels!  |
+---------------------------------+------------------------------------------------------------------------------------------------------------------+----------------------------------------+
```
Look at another message (this time we will go 3 deep on nesting to get the name of sender. <br>
Important note: the column “from” is a reserved SQL word. Use backticks when referring to this column (i.e. `from`). <br>
We will add Drill’s <b>substr</b> function to control the length of message.
```
0: jdbc:drill:> select data[1].id `messageID`,data[1].`from`.name `sender`,substr(data[1].message,1,40) `message` from dfs.`/Users/joeblue/sentiment/subway.json`;
+---------------------------------+----------------------------------------------+-------------------------------------------+
|            messageID            |                    sender                    |                  message                  |
+---------------------------------+----------------------------------------------+-------------------------------------------+
| 250313209090_10154064898059091  | Metropolitan Transportation Authority (MTA)  | Alternate service plan in effect tomorro  |
+---------------------------------+----------------------------------------------+-------------------------------------------+
1 row selected (0.15 seconds)
```
Working with one message at a time is good for exploring data, but we would like to work with more of the data at once. Use of the <b>flatten</b> function will come in handy. In the following query, we can show message ID, content and sender with one line per message. We have effectively turned a nested format into a semi-relational view.
```
0: jdbc:drill:zk=local> select t.flatdata.id `messageID`, substr(t.flatdata.message,1,50) `message`, 
t.flatdata.`from`.name `sender` from 
(select flatten(data) as flatdata from dfs.`/Users/joeblue/sentiment/subway.json`) t;

+---------------------------------+-----------------------------------------------------+----------------------------------------------+
|            messageID            |                       message                       |                    sender                    |
+---------------------------------+-----------------------------------------------------+----------------------------------------------+
| 250313209090_10154079915599091  | Big news for MTA Bridges and Tunnels!               | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10154064898059091  | Alternate service plan in effect tomorrow for Metr  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10154059515954091  | New MTA LIRR Concourse and redesign for MTA New Yo  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10154043349574091  | Two central MTA LIRR hubs to get major improvement  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10154004352224091  | Want to help keep New York safe?  Show your suppor  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153961971129091  | null                                                | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153960019754091  | The New Track Construction (NTC) machine makes its  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153959423079091  | null                                                | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153951499384091  | null                                                | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153951498429091  | null                                                | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153951472174091  | null                                                | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153943207744091  | Have professional experience in Drupal, Java, Orac  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153932719624091  | MTA Bridges and Tunnels at Orchard Beach tomorrow   | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153915154084091  | Operation Track Sweep Coming to All 469 Subway Sta  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153912643589091  | More countdown clocks and digital info screens!     | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153897327584091  | Nature at its best!                                 | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153886745694091  | Full Closure for 18 Months Starting in 2019.        | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153879485844091  | Monday is the day!                                  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153869000019091  | Governor Andrew Cuomo unveils design of reimagined  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153859595849091  | Introducing LIRR Getaways Guy!                      | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153856522434091  | Subway Service Advisory
Due to an electrical issue  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153841163254091  | Congrats to our 11 new MTAPD officers who graduate  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153837857609091  | We are holding a community meeting this Thursday,   | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153828693259091  | Early trains to start your holiday weekend. Make i  | Metropolitan Transportation Authority (MTA)  |
| 250313209090_10153811910764091  | Vintage rides along the Brighton Line this weekend  | Metropolitan Transportation Authority (MTA)  |
+---------------------------------+-----------------------------------------------------+----------------------------------------------+
25 rows selected (0.171 seconds)
```
Each message can have multiple likes and comments (which can generate even more reactions) and multiple levels of nesting. Using the flatten queries can help you dig into these structures.
```
0: jdbc:drill:zk=local> select t.flatdata.id `messageID`,
flatten(t.flatdata.likes.data) `likes` from 
(select flatten(data) as flatdata from dfs.`/Users/joeblue/sentiment/subway.json`) t limit 10;
+---------------------------------+-----------------------------------------------------+
|            messageID            |                        likes                        |
+---------------------------------+-----------------------------------------------------+
| 250313209090_10154079915599091  | {"id":"10211276701018432","name":"Cheryl Cuccio"}   |
| 250313209090_10154079915599091  | {"id":"10157645886905164","name":"Jenise Santana"}  |
| 250313209090_10154079915599091  | {"id":"945329802229998","name":"Tom Anderton"}      |
| 250313209090_10154079915599091  | {"id":"10207881301685381","name":"David Molnar"}    |
| 250313209090_10154079915599091  | {"id":"1086649124718665","name":"Arlene Rivera"}    |
| 250313209090_10154079915599091  | {"id":"10205317028643872","name":"Nareeza Nazam"}   |
| 250313209090_10154079915599091  | {"id":"1088091474544622","name":"Timothy Doran"}    |
| 250313209090_10154079915599091  | {"id":"1054017044696999","name":"Zane Amick"}       |
| 250313209090_10154079915599091  | {"id":"1664351853880399","name":"Maribel Ruiz"}     |
| 250313209090_10154079915599091  | {"id":"10208740614886427","name":"Olga De Jesus"}   |
+---------------------------------+-----------------------------------------------------+
10 rows selected (0.127 seconds)
```
Use nested <b>flatten</b> commands to generate the comments for each message. You can also extract the sender of the comments and other information. Note that when you explore real social data, you must be prepared for content that some might find offensive...
```
0: jdbc:drill:zk=local> select substr(f.message,1,70) `message`,
  substr(f.comment.message,1,50) `comment` from 
   (
    select 
      substr(t.flatdata.message,1,50) `message`,
      flatten(t.flatdata.comments.data) `comment`
      from 
      (select flatten(data) as flatdata from
        dfs.`/Users/joeblue/sentiment/subway.json`
      ) t
   ) f
 limit 50;
+-----------------------------------------------------+-----------------------------------------------------------+
|                       message                       |                          comment                          |
+-----------------------------------------------------+-----------------------------------------------------------+
| Big news for MTA Bridges and Tunnels!               | Great to see. But what about improved pedestrian a        |
| Big news for MTA Bridges and Tunnels!               | improving nothing this company is stealing our mon        |
| Big news for MTA Bridges and Tunnels!               | Stopless tolls good. Fancy lighting, nice but no a        |
| Big news for MTA Bridges and Tunnels!               | At this time of day bx9 buses heading to Riverdale        |
| Big news for MTA Bridges and Tunnels!               | MTA B15 bus number 6811 gets 👍🏽👍🏽👍🏽 from me on 10/  |
| Big news for MTA Bridges and Tunnels!               | MTA- This morning I was riding a BX3 bus at around        |
| Big news for MTA Bridges and Tunnels!               | Riverdale bound bx9 finally arrived but didn't hav        |
| Big news for MTA Bridges and Tunnels!               | the MFr's can't even get the signal fixed correctl        |
| Big news for MTA Bridges and Tunnels!               | I am sure it won't trigger the increase for tolls         |
| Big news for MTA Bridges and Tunnels!               | Why don't you guys fix the trains I'm sick of your        |
| Big news for MTA Bridges and Tunnels!               | What about your drivers, driving like maniac and r        |
| Big news for MTA Bridges and Tunnels!               | Downtown 4 trains so packed that the last stop any        |
| Big news for MTA Bridges and Tunnels!               | I live around east 180th in the bronx, how the hel        |
| Big news for MTA Bridges and Tunnels!               | I would like to know what is MTA'S policy on their        |
| Big news for MTA Bridges and Tunnels!               | Why is there always a delay on the Manhattan bound        |
| Big news for MTA Bridges and Tunnels!               | The Liberty Bridge is needed to connect Jersey Cit        |
| Big news for MTA Bridges and Tunnels!               | BX 24 Bus has been late every day for the last mon        |
| Big news for MTA Bridges and Tunnels!               | Why is Manhattan bound B train so slow between New        |
| Big news for MTA Bridges and Tunnels!               | what there are delays on the lexington line .at ru        |
| Big news for MTA Bridges and Tunnels!               | http://www.amny.com/transit/nyc-bus-service-report        |
| Big news for MTA Bridges and Tunnels!               | John Cowper they used my Flickr name!                     |
| Big news for MTA Bridges and Tunnels!               | Tolls are a joke so is the AC blasting on your bus        |
| Big news for MTA Bridges and Tunnels!               | Straight evil people                                      |
| Big news for MTA Bridges and Tunnels!               | Bunch of fucking crooks I swear to god!! Increase         |
| Big news for MTA Bridges and Tunnels!               | Great job.                                                |
| Alternate service plan in effect tomorrow for Metr  | Hey MTA, maybe instead of these useless updates yo        |
| Alternate service plan in effect tomorrow for Metr  | Nice progress on the subways; any plans to finish         |
| Alternate service plan in effect tomorrow for Metr  | 5 minutes, 2 more 12 and 2 28 and the next 9 is 4         |
| Alternate service plan in effect tomorrow for Metr  | Alternate service plan-everything is a disaster. F        |
| Alternate service plan in effect tomorrow for Metr  | Waited over 10 min for a Riverdale bound 9 bus...w        |
| Alternate service plan in effect tomorrow for Metr  | https://www.facebook.com/brightside/videos/3414454        |
| Alternate service plan in effect tomorrow for Metr  | Thanks for the communte homeService Change  Posted        |
| Alternate service plan in effect tomorrow for Metr  | https://www.facebook.com/djtwenty1/posts/115852600        |
| Alternate service plan in effect tomorrow for Metr  | Joey Singh                                                |
| Alternate service plan in effect tomorrow for Metr  | https://www.facebook.com/djtwenty1/posts/115852600        |
| Alternate service plan in effect tomorrow for Metr  | https://www.facebook.com/djtwenty1/posts/115852600        |
| Alternate service plan in effect tomorrow for Metr  | https://www.facebook.com/djtwenty1/posts/115852600        |
| Alternate service plan in effect tomorrow for Metr  | https://www.facebook.com/djtwenty1/posts/115852600        |
| Alternate service plan in effect tomorrow for Metr  | https://www.facebook.com/djtwenty1/posts/115852600        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | Last two day no 9:Am bx 24 bus in country club and        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | How about ya stop all this re-designing shyt and f        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | Let's put those new R179's to work to replace the         |
| New MTA LIRR Concourse and redesign for MTA New Yo  | Your service is simply unacceptable. 20 minutes an        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | Does this mean Gateway is making progress? Will th        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | Does this mean that if you work weekends it will t        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | Looks great unless it stays a homeless cesspool li        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | Delays  Posted: 09/29/2016  8:29AM 
| New MTA LIRR Concourse and redesign for MTA New Yo  | Wasn't there discussion of LIRR having a hub in lo        |
| New MTA LIRR Concourse and redesign for MTA New Yo  | December 2020... lol 😂                                   |
+-----------------------------------------------------+-----------------------------------------------------------+
50 rows selected (0.224 seconds)
```
## Shake Shack
People take hamburgers very seriously. Display some messages and comments from the Shake Shack feed…
```
0: jdbc:drill:zk=local> select substr(f.message,1,70) `message`,
  substr(f.comment.message,1,50) `comment` from 
   (
    select 
      substr(t.flatdata.message,1,50) `message`,
      flatten(t.flatdata.comments.data) `comment`
      from 
      (select flatten(data) as flatdata from
        dfs.`/Users/joeblue/sentiment/burger.json`
      ) t
   ) f
 limit 50;
+-----------------------------------------------------+-----------------------------------------------------+
|                       message                       |                       comment                       |
+-----------------------------------------------------+-----------------------------------------------------+
| WhoSay's Rob Gregory is having a delicious convers  | Will you guys ever put a shake shack in the coney   |
| WhoSay's Rob Gregory is having a delicious convers  | Hey                                                 |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Please stop sharing stuff like this!!!! I am starv  |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Yassssss Moises Castillo finally something i might  |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Crispy, sexy, crinkly, yummy.                       |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Omg! I will tear this thing up!                     |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  |                                                     |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | I need this 😍                                      |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Well that's a giant chicken sandwich.               |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Is that chicken?  Wow that looks goooood            |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | This is so, darn good.                              |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Cluck, cluck baby.                                  |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Osgood                                              |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  |                                                     |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Looks so good.                                      |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | That is what up i can get down on one them          |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Do you deliver!!!                                   |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | My fantasy                                          |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Samah Khasawneh T5aiale heeik Akel w 2a3deh 3ndk b  |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | DAAAAMN!!!!                                         |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | I can taste the crispy                              |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | looks like the Big Shack                            |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Kimberly Hitt I need this in my life                |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Mmmm                                                |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Jermaine Jenkins love me some Shake Shack!!         |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Mmmmmmm....                                         |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Margarita Nava Aguilar, una idea para el domingo.   |
| At the Flyers Game -Burger was cold - small portio  | Hi there! Would you mind sending us an email at sh  |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Every bite is the weekend.                          |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Steffi Loch                                         |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Im to old for this saying but in this case...DANGG  |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Yummy                                               |
| It's the freakin' weekend. Shot by IGer @orijusg.   |                                                     |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Need all of it now!!                                |
| It's the freakin' weekend. Shot by IGer @orijusg.   | yummy                                               |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Doooo you deliver!!!!!                              |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Kelly Mikunda would love one right about now!!      |
| It's the freakin' weekend. Shot by IGer @orijusg.   | 🍔🍻                                                |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Jem Collier 3 more days                             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Itzel Hernandez for us 😱💙💙💙                     |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Joshua Quisenberry lunch?                           |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Sohan Kalsi Parshva Shah Nikki Sharma soon!!!!      |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Regina Napier                                       |
| Crinkle, crinkle little star. Our classic crinkle   | Those fries look so good !! And I'm hungry right n  |
| Crinkle, crinkle little star. Our classic crinkle   | Fries are great where's the beef                    |
| Crinkle, crinkle little star. Our classic crinkle   | My tablet is drooling.                              |
| Crinkle, crinkle little star. Our classic crinkle   | Yummy fries                                         |
| Crinkle, crinkle little star. Our classic crinkle   | Yummy! Love me some crinkle cut fries!😝            |
| Crinkle, crinkle little star. Our classic crinkle   |                                                     |
| Crinkle, crinkle little star. Our classic crinkle   | Um munchies                                         |
+-----------------------------------------------------+-----------------------------------------------------+
50 rows selected (6.194 seconds)
```
You can use the <b>where</b> clause and <b>lower</b> function to filter results. In this query, we find the comments that contain the word “fries”, regardless of capitalization and display original message and comment. Cheese fries will start to sound good.
```
0: jdbc:drill:zk=local> select substr(f.message,1,70) `message`,
  substr(f.comment.message,1,50) `comment` from 
   (
    select 
      substr(t.flatdata.message,1,50) `message`,
      flatten(t.flatdata.comments.data) `comment`
      from 
      (select flatten(data) as flatdata from
        dfs.`/Users/joeblue/sentiment/burger.json`
      ) t
   ) f
 where lower(f.comment.message) like '%fries%'
 limit 50;
+-----------------------------------------------------+-----------------------------------------------------+
|                       message                       |                       comment                       |
+-----------------------------------------------------+-----------------------------------------------------+
| Crinkle, crinkle little star. Our classic crinkle   | Those fries look so good !! And I'm hungry right n  |
| Crinkle, crinkle little star. Our classic crinkle   | Fries are great where's the beef                    |
| Crinkle, crinkle little star. Our classic crinkle   | Yummy fries                                         |
| Crinkle, crinkle little star. Our classic crinkle   | Yummy! Love me some crinkle cut fries!😝            |
| Dive in it. Photo by IGer @papercupwords.           | Where's cheese fries?                               |
| Dive in it. Photo by IGer @papercupwords.           | Love crinkle cut fries.                             |
| Sun's out, buns out. Photo by grammer @keepingupwi  | I want cheese fries 😩 Kathleen Keen                |
+-----------------------------------------------------+-----------------------------------------------------+
```
Once you use flatten to split up the list of comments, you can use the traditional aggregrations like group by, count, etc.  
```
0: jdbc:drill:zk=local> select substr(x.message,1,50) `message`, count(*) `numComments`
   from ( select
      t.flatdata.message `message`,
      flatten(t.flatdata.comments.data) `comment`
      from 
      (select flatten(data) as flatdata from
        dfs.`/Users/joeblue/sentiment/burger.json`
      ) t
   ) x
   group by message;
+-----------------------------------------------------+--------------+
|                       message                       | numComments  |
+-----------------------------------------------------+--------------+
| WhoSay's Rob Gregory is having a delicious convers  | 2            |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | 25           |
| At the Flyers Game -Burger was cold - small portio  | 1            |
| It's the freakin' weekend. Shot by IGer @orijusg.   | 15           |
| Crinkle, crinkle little star. Our classic crinkle   | 9            |
| Just the two of us. Photo by IGer @sineunknown.     | 21           |
| I went to the one in Paramus New Jersey. NEVER NEV  | 1            |
| Shake Shack crew working at 6:07pm in JFK airport   | 1            |
| Dive in it. Photo by IGer @papercupwords.           | 25           |
| Congratulations to Shake Shack for winning the Chi  | 7            |
| Golden and Silver restaurants in Beirut as classif  | 2            |
| Make room for 'Shroom. Photo by IGer @revelationth  | 13           |
| We are almost 50% booked for our 24 hours of [soli  | 2            |
| Sun's out, buns out. Photo by grammer @keepingupwi  | 25           |
| Just in time for the Philadelphia Flyers home open  | 24           |
| Please please please open a location in Louisiana   | 2            |
| If there is vacancy job please, am not a lazy type  | 1            |
| Please build a store in Cherry Hill, NJ             | 1            |
+-----------------------------------------------------+--------------+
```
Seems like if we wanted to count the number of likes per comment on each message we would have to add another layer of nesting, then aggregate, but examination of the format shows that the field <b>like_count</b> already exists. 
```
0: jdbc:drill:zk=local> select substr(f.message,1,70) `message`,
  substr(f.comment.message,1,50) `comment`,
  f.comment.like_count `commentLikes` from 
   (
    select 
      substr(t.flatdata.message,1,50) `message`,
      flatten(t.flatdata.comments.data) `comment`
      from 
      (select flatten(data) as flatdata from
        dfs.`/Users/joeblue/sentiment/burger.json`
      ) t
   ) f
 limit 50;
+-----------------------------------------------------+-----------------------------------------------------+---------------+
|                       message                       |                       comment                       | commentLikes  |
+-----------------------------------------------------+-----------------------------------------------------+---------------+
| WhoSay's Rob Gregory is having a delicious convers  | Will you guys ever put a shake shack in the coney   | 0             |
| WhoSay's Rob Gregory is having a delicious convers  | Hey                                                 | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Please stop sharing stuff like this!!!! I am starv  | 1             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Yassssss Moises Castillo finally something i might  | 1             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Crispy, sexy, crinkly, yummy.                       | 1             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Omg! I will tear this thing up!                     | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  |                                                     | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | I need this 😍                                      | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Well that's a giant chicken sandwich.               | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Is that chicken?  Wow that looks goooood            | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | This is so, darn good.                              | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Cluck, cluck baby.                                  | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Osgood                                              | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  |                                                     | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Looks so good.                                      | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | That is what up i can get down on one them          | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Do you deliver!!!                                   | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | My fantasy                                          | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Samah Khasawneh T5aiale heeik Akel w 2a3deh 3ndk b  | 1             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | DAAAAMN!!!!                                         | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | I can taste the crispy                              | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | looks like the Big Shack                            | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Kimberly Hitt I need this in my life                | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Mmmm                                                | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Jermaine Jenkins love me some Shake Shack!!         | 1             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Mmmmmmm....                                         | 0             |
| Cluck yeah. Chick'n Shack shot by grammer @tinaaea  | Margarita Nava Aguilar, una idea para el domingo.   | 0             |
| At the Flyers Game -Burger was cold - small portio  | Hi there! Would you mind sending us an email at sh  | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Every bite is the weekend.                          | 1             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Steffi Loch                                         | 1             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Im to old for this saying but in this case...DANGG  | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Yummy                                               | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   |                                                     | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Need all of it now!!                                | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | yummy                                               | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Doooo you deliver!!!!!                              | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Kelly Mikunda would love one right about now!!      | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | 🍔🍻                                                | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Jem Collier 3 more days                             | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Itzel Hernandez for us 😱💙💙💙                     | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Joshua Quisenberry lunch?                           | 0             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Sohan Kalsi Parshva Shah Nikki Sharma soon!!!!      | 1             |
| It's the freakin' weekend. Shot by IGer @orijusg.   | Regina Napier                                       | 0             |
| Crinkle, crinkle little star. Our classic crinkle   | Those fries look so good !! And I'm hungry right n  | 1             |
| Crinkle, crinkle little star. Our classic crinkle   | Fries are great where's the beef                    | 0             |
| Crinkle, crinkle little star. Our classic crinkle   | My tablet is drooling.                              | 1             |
| Crinkle, crinkle little star. Our classic crinkle   | Yummy fries                                         | 0             |
| Crinkle, crinkle little star. Our classic crinkle   | Yummy! Love me some crinkle cut fries!😝            | 0             |
| Crinkle, crinkle little star. Our classic crinkle   |                                                     | 0             |
| Crinkle, crinkle little star. Our classic crinkle   | Um munchies                                         | 0             |
+-----------------------------------------------------+-----------------------------------------------------+---------------+
```


