In this article, the new way to gather official matches data will be presented, as well as some analysis oriented toward players.

## Gathering LoLEsports data

Following the shutdown of some important API from the lolesports website that allowed us to get hashes to gather data from official matches, the solution is to rely on Leaguepedia which has everything we need.

Here is an example of how to do it in Python.

We will use the [leaguepdia-parser](https://pypi.org/project/leaguepedia-parser/) package to gather what we need from Leaguepedia.

### Find the tournament


```python
import leaguepedia_parser

lp = leaguepedia_parser.LeaguepediaParser()
lp.get_tournament_regions()
```




    ['',
     'Africa',
     'Asia',
     'Brazil',
     'China',
     'CIS',
     'Europe',
     'International',
     'Japan',
     'Korea',
     'LAN',
     'LAS',
     'Latin America',
     'LMS',
     'MENA',
     'North America',
     'Oceania',
     'PCS',
     'SEA',
     'Turkey',
     'Unknown',
     'Vietnam',
     'Wildcard']



Pick your favorite region to get the list of tournaments in this region : 


```python
tournaments = lp.get_tournaments('Europe', year=2020)
[t["name"] for t in tournaments]
```




    ['LEC 2020 Spring',
     'LEC 2020 Spring Playoffs',
     'EU Face-Off 2020',
     'LEC 2020 Summer',
     'LEC 2020 Summer Playoffs']



We will create a custom method for the leaguepedia_parser to get only the information we need : 


```python
import types

def get_games_hashes(self, tournament_name=None, **kwargs):
    """
    Returns the list server, gameId and hashes of games played in a tournament.

    :param tournament_name
                Name of the tournament, which can be gotten from get_tournaments().
    :return:
                A list of game dictionaries.
    """
    games = self._cargoquery(tables='ScoreboardGames',
                             fields='Tournament = tournament, '
                                    'MatchHistory = match_history_url, ',
                             where="ScoreboardGames.Tournament='{}'".format(tournament_name),
                             order_by="ScoreboardGames.DateTime_UTC",
                             **kwargs)
    data = [
        {
            "tournament":game["tournament"],
            "server":game["match_history_url"].split("/")[5],
            "gameId":game["match_history_url"].split("/")[6].split("?gameHash=")[0],
            "hash":game["match_history_url"].split("/")[6].split("?gameHash=")[1],
        }
        for game in games
    ]
    return data

lp.get_games_hashes = types.MethodType(get_games_hashes, lp)
```

### Getting the hashes

Getting the hashes for LEC 2020 Summer : 


```python
games = lp.get_games_hashes(tournaments[3]['name'])
games[:3]
```




    [{'tournament': 'LEC 2020 Summer',
      'server': 'ESPORTSTMNT04',
      'gameId': '1230688',
      'hash': '25cb7e1966cbcdb5'},
     {'tournament': 'LEC 2020 Summer',
      'server': 'ESPORTSTMNT04',
      'gameId': '1220706',
      'hash': 'c3f45e5bb2a65c80'},
     {'tournament': 'LEC 2020 Summer',
      'server': 'ESPORTSTMNT04',
      'gameId': '1220728',
      'hash': '4bfd5c00f9292be3'}]



Requesting the match data from all those games : 


```python
import requests

base_match_history_stats_url = "https://acs.leagueoflegends.com/v1/stats/game/{}/{}?gameHash={}"
base_match_history_stats_timeline_url = "https://acs.leagueoflegends.com/v1/stats/game/{}/{}/timeline?gameHash={}"

all_games_data = []

for g in games:
    url = base_match_history_stats_url.format(g["server"],g["gameId"],g["hash"])
    timeline_url = base_match_history_stats_timeline_url.format(g["server"],g["gameId"],g["hash"])
    
    game_data = requests.get(url).json()
    game_data["timeline"] = requests.get(timeline_url).json()
    
    all_games_data.append(game_data)
```

If you get rate limited (errors 429), follow this guide : https://www.hextechdocs.dev/lol/esportsapi/13.esports-match-data#in-need-of-cookies

# Analysis

Once the data is ehre, let's crunch some numbers. First of all, you'll need to select a few specific pieces of information out of the whole match data. These functions will help for that : 


```python
# Team stats
# Get each team total damages dealt to champions
def get_team_damages_to_champions(g):
    t_dam = {100:0,200:0}
    for p in g["participants"]:
        t_dam[p["teamId"]] += p["stats"]["totalDamageDealtToChampions"]
    return t_dam

# Get the total kills of the team, as this is only available player by player
def get_team_kills(g):
    t_kills = {100:0,200:0}
    for p in g["participants"]:
        t_kills[p["teamId"]] += p["stats"]["kills"]
    return t_kills

# Participant stats
# Get the Gold Advantage of a player at the 15th minute
def ga_at_15(g, pId):
    g_at_15 = [p["totalGold"] for p in g["timeline"]["frames"][15]["participantFrames"].values() if p["participantId"] == pId][0]
    opp_pId = (pId+5)%10 if not pId == 5 else 10
    opp_g_at_15 = [p["totalGold"] for p in g["timeline"]["frames"][15]["participantFrames"].values() if p["participantId"] == opp_pId][0]
    return g_at_15 - opp_g_at_15

# Get the CS Difference of a player at the 15th minute
def cs_diff_at_15(g, pId):
    cs_at_15 = [p["minionsKilled"]+p["jungleMinionsKilled"] for p in g["timeline"]["frames"][15]["participantFrames"].values() if p["participantId"] == pId][0]
    opp_pId = (pId+5)%10 if not pId == 5 else 10
    opp_cs_at_15 = [p["minionsKilled"]+p["jungleMinionsKilled"] for p in g["timeline"]["frames"][15]["participantFrames"].values() if p["participantId"] == opp_pId][0]
    return cs_at_15 - opp_cs_at_15
```

Loading ddragon module to translate championId to their name. Note :  you need nest_asyncio only if you run this in a Jupyter Notebook.


```python
import nest_asyncio
nest_asyncio.apply()

import static_data
dd = static_data.ddragon()
```

    /usr/local/lib/python3.8/site-packages/aiohttp/helpers.py:107: DeprecationWarning: "@coroutine" decorator is deprecated since Python 3.8, use "async def" instead
      def noop(*args, **kwargs):  # type: ignore
    /usr/local/lib/python3.8/site-packages/aiohttp/connector.py:964: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      hosts = await asyncio.shield(self._resolve_host(
    /usr/local/lib/python3.8/site-packages/aiohttp/connector.py:964: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      hosts = await asyncio.shield(self._resolve_host(
    /usr/local/lib/python3.8/site-packages/aiohttp/connector.py:964: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      hosts = await asyncio.shield(self._resolve_host(
    /usr/local/lib/python3.8/site-packages/aiohttp/connector.py:964: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      hosts = await asyncio.shield(self._resolve_host(
    /usr/local/lib/python3.8/site-packages/aiohttp/connector.py:964: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      hosts = await asyncio.shield(self._resolve_host(
    /usr/local/lib/python3.8/site-packages/aiohttp/connector.py:964: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      hosts = await asyncio.shield(self._resolve_host(
    /usr/local/lib/python3.8/site-packages/aiohttp/locks.py:21: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      self._event = asyncio.Event(loop=loop)
    /usr/local/lib/python3.8/site-packages/aiohttp/locks.py:21: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      self._event = asyncio.Event(loop=loop)
    /usr/local/lib/python3.8/site-packages/aiohttp/locks.py:21: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      self._event = asyncio.Event(loop=loop)
    /usr/local/lib/python3.8/site-packages/aiohttp/locks.py:21: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      self._event = asyncio.Event(loop=loop)
    /usr/local/lib/python3.8/site-packages/aiohttp/locks.py:21: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      self._event = asyncio.Event(loop=loop)
    /usr/local/lib/python3.8/site-packages/aiohttp/locks.py:21: DeprecationWarning: The loop argument is deprecated since Python 3.8, and scheduled for removal in Python 3.10.
      self._event = asyncio.Event(loop=loop)


Gather stats from each player in each game.


```python
players_stats = []

for g in all_games_data:
    
    # Get the teams stats
    teams_damages_to_champions = get_team_damages_to_champions(g)
    teams_kills = get_team_kills(g)
    
    
    # Translate participantId to the player's name
    pId_to_name = {}
    for p in g["participantIdentities"]:
        pId_to_name[p["participantId"]] = p["player"]["summonerName"]
    
    for p in g["participants"]:
        players_stats.append({
            "name":pId_to_name[p["participantId"]],
            "champion":dd.getChampion(p["championId"]).name,
            "damage_share":p["stats"]["totalDamageDealtToChampions"] / teams_damages_to_champions[p["teamId"]],
            "kill_participation":(p["stats"]["kills"] + p["stats"]["assists"]) / teams_kills[p["teamId"]] if teams_kills[p["teamId"]] > 0 else 0,
            "cs_diff_at_15":cs_diff_at_15(g, p["participantId"]),
            "ga_at_15":ga_at_15(g, p["participantId"]),
            "kills":p["stats"]["kills"],
            "deaths":p["stats"]["deaths"],
            "assists":p["stats"]["assists"],
            "vision_score":p["stats"]["visionScore"],
            "damage_per_gold":p["stats"]["totalDamageDealtToChampions"] / p["stats"]["goldEarned"],
            "win":p["stats"]["win"]
        })
```

Put the stats into a pandas DataFrame.


```python
import pandas as pd
df = pd.DataFrame(players_stats)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>name</th>
      <th>champion</th>
      <th>damage_share</th>
      <th>kill_participation</th>
      <th>cs_diff_at_15</th>
      <th>ga_at_15</th>
      <th>kills</th>
      <th>deaths</th>
      <th>assists</th>
      <th>vision_score</th>
      <th>damage_per_gold</th>
      <th>win</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>G2 Wunder</td>
      <td>Mordekaiser</td>
      <td>0.207249</td>
      <td>0.454545</td>
      <td>8</td>
      <td>955</td>
      <td>6</td>
      <td>1</td>
      <td>4</td>
      <td>32</td>
      <td>0.877160</td>
      <td>True</td>
    </tr>
    <tr>
      <th>1</th>
      <td>G2 Jankos</td>
      <td>Rek'Sai</td>
      <td>0.103866</td>
      <td>0.318182</td>
      <td>10</td>
      <td>-67</td>
      <td>1</td>
      <td>2</td>
      <td>6</td>
      <td>64</td>
      <td>0.540224</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>G2 Caps</td>
      <td>Zoe</td>
      <td>0.184098</td>
      <td>0.636364</td>
      <td>-30</td>
      <td>37</td>
      <td>5</td>
      <td>1</td>
      <td>9</td>
      <td>34</td>
      <td>0.844762</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>G2 Perkz</td>
      <td>Varus</td>
      <td>0.421698</td>
      <td>0.818182</td>
      <td>37</td>
      <td>1680</td>
      <td>9</td>
      <td>0</td>
      <td>9</td>
      <td>26</td>
      <td>1.516857</td>
      <td>True</td>
    </tr>
    <tr>
      <th>4</th>
      <td>G2 Mikyx</td>
      <td>Tahm Kench</td>
      <td>0.083089</td>
      <td>0.545455</td>
      <td>8</td>
      <td>650</td>
      <td>1</td>
      <td>1</td>
      <td>11</td>
      <td>80</td>
      <td>0.566123</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



Group the mean of each stat and the number of games they played in.


```python
df_results = pd.concat([df.groupby("name").mean(), df.groupby("name").agg(Games=("kills","count"))], axis=1)
df_results
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>damage_share</th>
      <th>kill_participation</th>
      <th>cs_diff_at_15</th>
      <th>ga_at_15</th>
      <th>kills</th>
      <th>deaths</th>
      <th>assists</th>
      <th>vision_score</th>
      <th>damage_per_gold</th>
      <th>win</th>
      <th>Games</th>
    </tr>
    <tr>
      <th>name</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>FNC Bwipo</th>
      <td>0.223127</td>
      <td>0.593547</td>
      <td>-9.722222</td>
      <td>-194.111111</td>
      <td>2.722222</td>
      <td>3.833333</td>
      <td>5.166667</td>
      <td>32.777778</td>
      <td>1.136960</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Hylissang</th>
      <td>0.083543</td>
      <td>0.596570</td>
      <td>-5.888889</td>
      <td>140.444444</td>
      <td>0.833333</td>
      <td>4.555556</td>
      <td>7.055556</td>
      <td>84.555556</td>
      <td>0.667025</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Nemesis</th>
      <td>0.247389</td>
      <td>0.562223</td>
      <td>-3.166667</td>
      <td>-190.666667</td>
      <td>2.722222</td>
      <td>2.388889</td>
      <td>4.611111</td>
      <td>32.777778</td>
      <td>1.105412</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Rekkles</th>
      <td>0.258534</td>
      <td>0.775209</td>
      <td>5.777778</td>
      <td>72.444444</td>
      <td>3.055556</td>
      <td>1.333333</td>
      <td>6.333333</td>
      <td>36.944444</td>
      <td>1.107452</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Selfmade</th>
      <td>0.187407</td>
      <td>0.710503</td>
      <td>11.444444</td>
      <td>201.055556</td>
      <td>3.000000</td>
      <td>2.444444</td>
      <td>5.500000</td>
      <td>44.611111</td>
      <td>0.961258</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 Caps</th>
      <td>0.293738</td>
      <td>0.700959</td>
      <td>2.333333</td>
      <td>384.222222</td>
      <td>4.777778</td>
      <td>2.111111</td>
      <td>5.277778</td>
      <td>31.500000</td>
      <td>1.289828</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 Jankos</th>
      <td>0.135096</td>
      <td>0.701737</td>
      <td>0.722222</td>
      <td>4.722222</td>
      <td>2.166667</td>
      <td>2.666667</td>
      <td>6.833333</td>
      <td>50.777778</td>
      <td>0.795349</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 Mikyx</th>
      <td>0.085227</td>
      <td>0.574378</td>
      <td>8.388889</td>
      <td>217.055556</td>
      <td>0.777778</td>
      <td>2.944444</td>
      <td>7.388889</td>
      <td>90.888889</td>
      <td>0.647563</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 P1noy</th>
      <td>0.147956</td>
      <td>0.560606</td>
      <td>-67.500000</td>
      <td>-1493.500000</td>
      <td>3.500000</td>
      <td>2.500000</td>
      <td>5.500000</td>
      <td>43.500000</td>
      <td>1.070342</td>
      <td>0.500000</td>
      <td>2</td>
    </tr>
    <tr>
      <th>G2 Perkz</th>
      <td>0.279655</td>
      <td>0.557734</td>
      <td>-3.437500</td>
      <td>-260.187500</td>
      <td>3.875000</td>
      <td>2.562500</td>
      <td>4.812500</td>
      <td>37.000000</td>
      <td>1.195827</td>
      <td>0.625000</td>
      <td>16</td>
    </tr>
    <tr>
      <th>G2 Wunder</th>
      <td>0.220917</td>
      <td>0.571255</td>
      <td>12.277778</td>
      <td>404.777778</td>
      <td>2.555556</td>
      <td>2.722222</td>
      <td>5.055556</td>
      <td>31.277778</td>
      <td>1.105366</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Carzzy</th>
      <td>0.252405</td>
      <td>0.665892</td>
      <td>-29.722222</td>
      <td>-550.055556</td>
      <td>3.055556</td>
      <td>1.944444</td>
      <td>6.722222</td>
      <td>44.888889</td>
      <td>1.336119</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Humanoid</th>
      <td>0.308612</td>
      <td>0.676741</td>
      <td>8.666667</td>
      <td>269.055556</td>
      <td>4.388889</td>
      <td>2.777778</td>
      <td>5.500000</td>
      <td>31.888889</td>
      <td>1.368744</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Kaiser</th>
      <td>0.104116</td>
      <td>0.683497</td>
      <td>19.555556</td>
      <td>368.888889</td>
      <td>1.333333</td>
      <td>2.000000</td>
      <td>8.444444</td>
      <td>79.833333</td>
      <td>0.794702</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Orome</th>
      <td>0.202742</td>
      <td>0.556667</td>
      <td>-3.777778</td>
      <td>173.333333</td>
      <td>2.777778</td>
      <td>1.833333</td>
      <td>5.222222</td>
      <td>28.666667</td>
      <td>1.038037</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Shad0w</th>
      <td>0.132126</td>
      <td>0.716630</td>
      <td>-8.222222</td>
      <td>-109.666667</td>
      <td>2.611111</td>
      <td>2.555556</td>
      <td>7.222222</td>
      <td>44.055556</td>
      <td>0.794622</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Dan Dan</th>
      <td>0.232844</td>
      <td>0.593369</td>
      <td>-3.666667</td>
      <td>-257.555556</td>
      <td>2.222222</td>
      <td>2.000000</td>
      <td>3.388889</td>
      <td>30.833333</td>
      <td>1.054232</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Doss</th>
      <td>0.071463</td>
      <td>0.553586</td>
      <td>-1.230769</td>
      <td>19.692308</td>
      <td>0.230769</td>
      <td>3.153846</td>
      <td>5.769231</td>
      <td>81.538462</td>
      <td>0.539777</td>
      <td>0.384615</td>
      <td>13</td>
    </tr>
    <tr>
      <th>MSF FEBIVEN</th>
      <td>0.285648</td>
      <td>0.647716</td>
      <td>-0.388889</td>
      <td>-303.555556</td>
      <td>2.444444</td>
      <td>2.611111</td>
      <td>4.166667</td>
      <td>32.166667</td>
      <td>1.293193</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Kobbe</th>
      <td>0.269913</td>
      <td>0.641566</td>
      <td>-2.722222</td>
      <td>-4.277778</td>
      <td>3.222222</td>
      <td>1.833333</td>
      <td>3.888889</td>
      <td>35.611111</td>
      <td>1.092888</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Razork</th>
      <td>0.137768</td>
      <td>0.723598</td>
      <td>7.222222</td>
      <td>83.388889</td>
      <td>2.000000</td>
      <td>3.388889</td>
      <td>5.111111</td>
      <td>52.555556</td>
      <td>0.777673</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF denyk</th>
      <td>0.079974</td>
      <td>0.631178</td>
      <td>2.000000</td>
      <td>-363.000000</td>
      <td>0.800000</td>
      <td>4.600000</td>
      <td>6.600000</td>
      <td>98.800000</td>
      <td>0.739018</td>
      <td>0.400000</td>
      <td>5</td>
    </tr>
    <tr>
      <th>OG Alphari</th>
      <td>0.248336</td>
      <td>0.631274</td>
      <td>17.500000</td>
      <td>388.944444</td>
      <td>2.055556</td>
      <td>2.388889</td>
      <td>4.055556</td>
      <td>35.555556</td>
      <td>1.133699</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>OG Destiny</th>
      <td>0.059221</td>
      <td>0.602118</td>
      <td>-1.666667</td>
      <td>-47.111111</td>
      <td>0.666667</td>
      <td>2.000000</td>
      <td>5.888889</td>
      <td>91.888889</td>
      <td>0.479166</td>
      <td>0.444444</td>
      <td>9</td>
    </tr>
    <tr>
      <th>OG Jactroll</th>
      <td>0.068711</td>
      <td>0.688606</td>
      <td>-5.555556</td>
      <td>-293.555556</td>
      <td>0.222222</td>
      <td>3.777778</td>
      <td>5.444444</td>
      <td>97.222222</td>
      <td>0.608410</td>
      <td>0.222222</td>
      <td>9</td>
    </tr>
    <tr>
      <th>OG Nukeduck</th>
      <td>0.231971</td>
      <td>0.704894</td>
      <td>-5.111111</td>
      <td>-249.500000</td>
      <td>2.055556</td>
      <td>2.333333</td>
      <td>4.444444</td>
      <td>35.888889</td>
      <td>1.088042</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>OG Upset</th>
      <td>0.321765</td>
      <td>0.723173</td>
      <td>9.277778</td>
      <td>11.055556</td>
      <td>3.388889</td>
      <td>1.222222</td>
      <td>3.833333</td>
      <td>47.500000</td>
      <td>1.371337</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>OG Xerxe</th>
      <td>0.133963</td>
      <td>0.688985</td>
      <td>0.333333</td>
      <td>-125.555556</td>
      <td>1.555556</td>
      <td>2.166667</td>
      <td>4.833333</td>
      <td>64.166667</td>
      <td>0.802195</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Finn</th>
      <td>0.239404</td>
      <td>0.599334</td>
      <td>-4.333333</td>
      <td>230.333333</td>
      <td>1.888889</td>
      <td>2.055556</td>
      <td>6.111111</td>
      <td>41.833333</td>
      <td>1.118791</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Hans sama</th>
      <td>0.282284</td>
      <td>0.656932</td>
      <td>10.333333</td>
      <td>525.111111</td>
      <td>4.444444</td>
      <td>1.722222</td>
      <td>4.555556</td>
      <td>42.444444</td>
      <td>1.188017</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Inspired</th>
      <td>0.098220</td>
      <td>0.700881</td>
      <td>-4.833333</td>
      <td>102.611111</td>
      <td>1.666667</td>
      <td>1.000000</td>
      <td>8.000000</td>
      <td>57.611111</td>
      <td>0.567715</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Larssen</th>
      <td>0.312793</td>
      <td>0.720134</td>
      <td>11.388889</td>
      <td>538.388889</td>
      <td>4.777778</td>
      <td>1.444444</td>
      <td>4.666667</td>
      <td>30.222222</td>
      <td>1.247148</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Vander</th>
      <td>0.067299</td>
      <td>0.595965</td>
      <td>-5.555556</td>
      <td>-98.277778</td>
      <td>0.666667</td>
      <td>0.944444</td>
      <td>7.500000</td>
      <td>82.055556</td>
      <td>0.569031</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>S04 Abbedagge</th>
      <td>0.285159</td>
      <td>0.706940</td>
      <td>0.277778</td>
      <td>3.611111</td>
      <td>3.500000</td>
      <td>2.111111</td>
      <td>4.277778</td>
      <td>40.833333</td>
      <td>1.289298</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>S04 Dreams</th>
      <td>0.081275</td>
      <td>0.671343</td>
      <td>0.266667</td>
      <td>-36.800000</td>
      <td>0.866667</td>
      <td>2.400000</td>
      <td>7.400000</td>
      <td>84.800000</td>
      <td>0.678752</td>
      <td>0.533333</td>
      <td>15</td>
    </tr>
    <tr>
      <th>S04 Gilius</th>
      <td>0.137355</td>
      <td>0.736826</td>
      <td>5.833333</td>
      <td>442.083333</td>
      <td>4.000000</td>
      <td>2.083333</td>
      <td>6.250000</td>
      <td>58.000000</td>
      <td>0.814762</td>
      <td>0.666667</td>
      <td>12</td>
    </tr>
    <tr>
      <th>S04 Innaxe</th>
      <td>0.384667</td>
      <td>0.858586</td>
      <td>-24.666667</td>
      <td>-1184.000000</td>
      <td>2.666667</td>
      <td>3.000000</td>
      <td>1.666667</td>
      <td>39.333333</td>
      <td>1.650448</td>
      <td>0.000000</td>
      <td>3</td>
    </tr>
    <tr>
      <th>S04 Lurox</th>
      <td>0.097811</td>
      <td>0.741703</td>
      <td>-7.666667</td>
      <td>-228.333333</td>
      <td>0.666667</td>
      <td>2.500000</td>
      <td>2.000000</td>
      <td>56.500000</td>
      <td>0.522445</td>
      <td>0.000000</td>
      <td>6</td>
    </tr>
    <tr>
      <th>S04 Neon</th>
      <td>0.293243</td>
      <td>0.727295</td>
      <td>-1.600000</td>
      <td>-83.800000</td>
      <td>2.600000</td>
      <td>2.133333</td>
      <td>6.266667</td>
      <td>39.266667</td>
      <td>1.353045</td>
      <td>0.533333</td>
      <td>15</td>
    </tr>
    <tr>
      <th>S04 Nukes</th>
      <td>0.062412</td>
      <td>0.797980</td>
      <td>5.000000</td>
      <td>199.000000</td>
      <td>0.666667</td>
      <td>2.666667</td>
      <td>3.000000</td>
      <td>106.666667</td>
      <td>0.403134</td>
      <td>0.000000</td>
      <td>3</td>
    </tr>
    <tr>
      <th>S04 Odoamne</th>
      <td>0.204055</td>
      <td>0.665810</td>
      <td>4.444444</td>
      <td>-25.500000</td>
      <td>1.500000</td>
      <td>2.555556</td>
      <td>6.111111</td>
      <td>33.000000</td>
      <td>1.016403</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK Crownshot</th>
      <td>0.305696</td>
      <td>0.682608</td>
      <td>11.888889</td>
      <td>264.555556</td>
      <td>3.555556</td>
      <td>1.833333</td>
      <td>4.833333</td>
      <td>40.111111</td>
      <td>1.248497</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK Jenax</th>
      <td>0.231644</td>
      <td>0.560635</td>
      <td>-5.222222</td>
      <td>-460.388889</td>
      <td>2.611111</td>
      <td>2.611111</td>
      <td>4.444444</td>
      <td>32.277778</td>
      <td>1.120304</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK LIMIT</th>
      <td>0.089615</td>
      <td>0.711859</td>
      <td>-8.055556</td>
      <td>-138.944444</td>
      <td>0.888889</td>
      <td>2.500000</td>
      <td>7.833333</td>
      <td>95.555556</td>
      <td>0.699940</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK Trick</th>
      <td>0.114979</td>
      <td>0.642871</td>
      <td>0.222222</td>
      <td>-151.333333</td>
      <td>1.666667</td>
      <td>2.833333</td>
      <td>6.333333</td>
      <td>40.666667</td>
      <td>0.685385</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK ZaZee</th>
      <td>0.258067</td>
      <td>0.749928</td>
      <td>-4.166667</td>
      <td>-79.666667</td>
      <td>3.333333</td>
      <td>2.166667</td>
      <td>5.666667</td>
      <td>33.500000</td>
      <td>1.087049</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Cabochard</th>
      <td>0.195603</td>
      <td>0.555988</td>
      <td>-6.777778</td>
      <td>-393.833333</td>
      <td>1.944444</td>
      <td>2.277778</td>
      <td>4.000000</td>
      <td>36.111111</td>
      <td>0.942578</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Comp</th>
      <td>0.337604</td>
      <td>0.788173</td>
      <td>2.222222</td>
      <td>94.888889</td>
      <td>4.000000</td>
      <td>2.277778</td>
      <td>4.555556</td>
      <td>45.722222</td>
      <td>1.281572</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Labrov</th>
      <td>0.073434</td>
      <td>0.792618</td>
      <td>0.611111</td>
      <td>-145.833333</td>
      <td>0.388889</td>
      <td>2.611111</td>
      <td>8.222222</td>
      <td>103.111111</td>
      <td>0.552762</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Milica</th>
      <td>0.290508</td>
      <td>0.692663</td>
      <td>-6.166667</td>
      <td>-254.000000</td>
      <td>3.111111</td>
      <td>1.888889</td>
      <td>4.388889</td>
      <td>45.000000</td>
      <td>1.123236</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Nji</th>
      <td>0.092451</td>
      <td>0.572866</td>
      <td>-9.666667</td>
      <td>-6.222222</td>
      <td>0.555556</td>
      <td>3.111111</td>
      <td>4.777778</td>
      <td>60.666667</td>
      <td>0.463280</td>
      <td>0.444444</td>
      <td>9</td>
    </tr>
    <tr>
      <th>VIT Skeanz</th>
      <td>0.113250</td>
      <td>0.673732</td>
      <td>-7.444444</td>
      <td>-270.666667</td>
      <td>2.000000</td>
      <td>2.666667</td>
      <td>5.888889</td>
      <td>50.111111</td>
      <td>0.686650</td>
      <td>0.333333</td>
      <td>9</td>
    </tr>
    <tr>
      <th>XL Caedrel</th>
      <td>0.104300</td>
      <td>0.608204</td>
      <td>0.333333</td>
      <td>-85.388889</td>
      <td>1.444444</td>
      <td>2.222222</td>
      <td>4.055556</td>
      <td>52.111111</td>
      <td>0.576975</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Kryze</th>
      <td>0.241151</td>
      <td>0.546163</td>
      <td>-0.722222</td>
      <td>134.000000</td>
      <td>1.888889</td>
      <td>2.444444</td>
      <td>2.944444</td>
      <td>30.111111</td>
      <td>0.992337</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Patrik</th>
      <td>0.332169</td>
      <td>0.704439</td>
      <td>8.944444</td>
      <td>250.666667</td>
      <td>3.611111</td>
      <td>2.055556</td>
      <td>3.500000</td>
      <td>42.000000</td>
      <td>1.320411</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Special</th>
      <td>0.233506</td>
      <td>0.608768</td>
      <td>-3.666667</td>
      <td>-117.888889</td>
      <td>2.000000</td>
      <td>2.500000</td>
      <td>3.833333</td>
      <td>35.722222</td>
      <td>0.950198</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Tore</th>
      <td>0.088873</td>
      <td>0.677933</td>
      <td>-6.166667</td>
      <td>-88.888889</td>
      <td>0.166667</td>
      <td>2.388889</td>
      <td>6.388889</td>
      <td>92.500000</td>
      <td>0.662340</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
  </tbody>
</table>
</div>



Then find out who is the best support


```python
df_results.sort_values("vision_score", ascending=False)[:5]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>damage_share</th>
      <th>kill_participation</th>
      <th>cs_diff_at_15</th>
      <th>ga_at_15</th>
      <th>kills</th>
      <th>deaths</th>
      <th>assists</th>
      <th>vision_score</th>
      <th>damage_per_gold</th>
      <th>win</th>
      <th>Games</th>
    </tr>
    <tr>
      <th>name</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>S04 Nukes</th>
      <td>0.062412</td>
      <td>0.797980</td>
      <td>5.000000</td>
      <td>199.000000</td>
      <td>0.666667</td>
      <td>2.666667</td>
      <td>3.000000</td>
      <td>106.666667</td>
      <td>0.403134</td>
      <td>0.000000</td>
      <td>3</td>
    </tr>
    <tr>
      <th>VIT Labrov</th>
      <td>0.073434</td>
      <td>0.792618</td>
      <td>0.611111</td>
      <td>-145.833333</td>
      <td>0.388889</td>
      <td>2.611111</td>
      <td>8.222222</td>
      <td>103.111111</td>
      <td>0.552762</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF denyk</th>
      <td>0.079974</td>
      <td>0.631178</td>
      <td>2.000000</td>
      <td>-363.000000</td>
      <td>0.800000</td>
      <td>4.600000</td>
      <td>6.600000</td>
      <td>98.800000</td>
      <td>0.739018</td>
      <td>0.400000</td>
      <td>5</td>
    </tr>
    <tr>
      <th>OG Jactroll</th>
      <td>0.068711</td>
      <td>0.688606</td>
      <td>-5.555556</td>
      <td>-293.555556</td>
      <td>0.222222</td>
      <td>3.777778</td>
      <td>5.444444</td>
      <td>97.222222</td>
      <td>0.608410</td>
      <td>0.222222</td>
      <td>9</td>
    </tr>
    <tr>
      <th>SK LIMIT</th>
      <td>0.089615</td>
      <td>0.711859</td>
      <td>-8.055556</td>
      <td>-138.944444</td>
      <td>0.888889</td>
      <td>2.500000</td>
      <td>7.833333</td>
      <td>95.555556</td>
      <td>0.699940</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
  </tbody>
</table>
</div>



You can also filter out players that didn't played a lot


```python
df_results[df_results["Games"] > 9]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>damage_share</th>
      <th>kill_participation</th>
      <th>cs_diff_at_15</th>
      <th>ga_at_15</th>
      <th>kills</th>
      <th>deaths</th>
      <th>assists</th>
      <th>vision_score</th>
      <th>damage_per_gold</th>
      <th>win</th>
      <th>Games</th>
    </tr>
    <tr>
      <th>name</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>FNC Bwipo</th>
      <td>0.223127</td>
      <td>0.593547</td>
      <td>-9.722222</td>
      <td>-194.111111</td>
      <td>2.722222</td>
      <td>3.833333</td>
      <td>5.166667</td>
      <td>32.777778</td>
      <td>1.136960</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Hylissang</th>
      <td>0.083543</td>
      <td>0.596570</td>
      <td>-5.888889</td>
      <td>140.444444</td>
      <td>0.833333</td>
      <td>4.555556</td>
      <td>7.055556</td>
      <td>84.555556</td>
      <td>0.667025</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Nemesis</th>
      <td>0.247389</td>
      <td>0.562223</td>
      <td>-3.166667</td>
      <td>-190.666667</td>
      <td>2.722222</td>
      <td>2.388889</td>
      <td>4.611111</td>
      <td>32.777778</td>
      <td>1.105412</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Rekkles</th>
      <td>0.258534</td>
      <td>0.775209</td>
      <td>5.777778</td>
      <td>72.444444</td>
      <td>3.055556</td>
      <td>1.333333</td>
      <td>6.333333</td>
      <td>36.944444</td>
      <td>1.107452</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>FNC Selfmade</th>
      <td>0.187407</td>
      <td>0.710503</td>
      <td>11.444444</td>
      <td>201.055556</td>
      <td>3.000000</td>
      <td>2.444444</td>
      <td>5.500000</td>
      <td>44.611111</td>
      <td>0.961258</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 Caps</th>
      <td>0.293738</td>
      <td>0.700959</td>
      <td>2.333333</td>
      <td>384.222222</td>
      <td>4.777778</td>
      <td>2.111111</td>
      <td>5.277778</td>
      <td>31.500000</td>
      <td>1.289828</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 Jankos</th>
      <td>0.135096</td>
      <td>0.701737</td>
      <td>0.722222</td>
      <td>4.722222</td>
      <td>2.166667</td>
      <td>2.666667</td>
      <td>6.833333</td>
      <td>50.777778</td>
      <td>0.795349</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 Mikyx</th>
      <td>0.085227</td>
      <td>0.574378</td>
      <td>8.388889</td>
      <td>217.055556</td>
      <td>0.777778</td>
      <td>2.944444</td>
      <td>7.388889</td>
      <td>90.888889</td>
      <td>0.647563</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>G2 Perkz</th>
      <td>0.279655</td>
      <td>0.557734</td>
      <td>-3.437500</td>
      <td>-260.187500</td>
      <td>3.875000</td>
      <td>2.562500</td>
      <td>4.812500</td>
      <td>37.000000</td>
      <td>1.195827</td>
      <td>0.625000</td>
      <td>16</td>
    </tr>
    <tr>
      <th>G2 Wunder</th>
      <td>0.220917</td>
      <td>0.571255</td>
      <td>12.277778</td>
      <td>404.777778</td>
      <td>2.555556</td>
      <td>2.722222</td>
      <td>5.055556</td>
      <td>31.277778</td>
      <td>1.105366</td>
      <td>0.611111</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Carzzy</th>
      <td>0.252405</td>
      <td>0.665892</td>
      <td>-29.722222</td>
      <td>-550.055556</td>
      <td>3.055556</td>
      <td>1.944444</td>
      <td>6.722222</td>
      <td>44.888889</td>
      <td>1.336119</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Humanoid</th>
      <td>0.308612</td>
      <td>0.676741</td>
      <td>8.666667</td>
      <td>269.055556</td>
      <td>4.388889</td>
      <td>2.777778</td>
      <td>5.500000</td>
      <td>31.888889</td>
      <td>1.368744</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Kaiser</th>
      <td>0.104116</td>
      <td>0.683497</td>
      <td>19.555556</td>
      <td>368.888889</td>
      <td>1.333333</td>
      <td>2.000000</td>
      <td>8.444444</td>
      <td>79.833333</td>
      <td>0.794702</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Orome</th>
      <td>0.202742</td>
      <td>0.556667</td>
      <td>-3.777778</td>
      <td>173.333333</td>
      <td>2.777778</td>
      <td>1.833333</td>
      <td>5.222222</td>
      <td>28.666667</td>
      <td>1.038037</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MAD Shad0w</th>
      <td>0.132126</td>
      <td>0.716630</td>
      <td>-8.222222</td>
      <td>-109.666667</td>
      <td>2.611111</td>
      <td>2.555556</td>
      <td>7.222222</td>
      <td>44.055556</td>
      <td>0.794622</td>
      <td>0.666667</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Dan Dan</th>
      <td>0.232844</td>
      <td>0.593369</td>
      <td>-3.666667</td>
      <td>-257.555556</td>
      <td>2.222222</td>
      <td>2.000000</td>
      <td>3.388889</td>
      <td>30.833333</td>
      <td>1.054232</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Doss</th>
      <td>0.071463</td>
      <td>0.553586</td>
      <td>-1.230769</td>
      <td>19.692308</td>
      <td>0.230769</td>
      <td>3.153846</td>
      <td>5.769231</td>
      <td>81.538462</td>
      <td>0.539777</td>
      <td>0.384615</td>
      <td>13</td>
    </tr>
    <tr>
      <th>MSF FEBIVEN</th>
      <td>0.285648</td>
      <td>0.647716</td>
      <td>-0.388889</td>
      <td>-303.555556</td>
      <td>2.444444</td>
      <td>2.611111</td>
      <td>4.166667</td>
      <td>32.166667</td>
      <td>1.293193</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Kobbe</th>
      <td>0.269913</td>
      <td>0.641566</td>
      <td>-2.722222</td>
      <td>-4.277778</td>
      <td>3.222222</td>
      <td>1.833333</td>
      <td>3.888889</td>
      <td>35.611111</td>
      <td>1.092888</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>MSF Razork</th>
      <td>0.137768</td>
      <td>0.723598</td>
      <td>7.222222</td>
      <td>83.388889</td>
      <td>2.000000</td>
      <td>3.388889</td>
      <td>5.111111</td>
      <td>52.555556</td>
      <td>0.777673</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>OG Alphari</th>
      <td>0.248336</td>
      <td>0.631274</td>
      <td>17.500000</td>
      <td>388.944444</td>
      <td>2.055556</td>
      <td>2.388889</td>
      <td>4.055556</td>
      <td>35.555556</td>
      <td>1.133699</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>OG Nukeduck</th>
      <td>0.231971</td>
      <td>0.704894</td>
      <td>-5.111111</td>
      <td>-249.500000</td>
      <td>2.055556</td>
      <td>2.333333</td>
      <td>4.444444</td>
      <td>35.888889</td>
      <td>1.088042</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>OG Upset</th>
      <td>0.321765</td>
      <td>0.723173</td>
      <td>9.277778</td>
      <td>11.055556</td>
      <td>3.388889</td>
      <td>1.222222</td>
      <td>3.833333</td>
      <td>47.500000</td>
      <td>1.371337</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>OG Xerxe</th>
      <td>0.133963</td>
      <td>0.688985</td>
      <td>0.333333</td>
      <td>-125.555556</td>
      <td>1.555556</td>
      <td>2.166667</td>
      <td>4.833333</td>
      <td>64.166667</td>
      <td>0.802195</td>
      <td>0.333333</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Finn</th>
      <td>0.239404</td>
      <td>0.599334</td>
      <td>-4.333333</td>
      <td>230.333333</td>
      <td>1.888889</td>
      <td>2.055556</td>
      <td>6.111111</td>
      <td>41.833333</td>
      <td>1.118791</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Hans sama</th>
      <td>0.282284</td>
      <td>0.656932</td>
      <td>10.333333</td>
      <td>525.111111</td>
      <td>4.444444</td>
      <td>1.722222</td>
      <td>4.555556</td>
      <td>42.444444</td>
      <td>1.188017</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Inspired</th>
      <td>0.098220</td>
      <td>0.700881</td>
      <td>-4.833333</td>
      <td>102.611111</td>
      <td>1.666667</td>
      <td>1.000000</td>
      <td>8.000000</td>
      <td>57.611111</td>
      <td>0.567715</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Larssen</th>
      <td>0.312793</td>
      <td>0.720134</td>
      <td>11.388889</td>
      <td>538.388889</td>
      <td>4.777778</td>
      <td>1.444444</td>
      <td>4.666667</td>
      <td>30.222222</td>
      <td>1.247148</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>RGE Vander</th>
      <td>0.067299</td>
      <td>0.595965</td>
      <td>-5.555556</td>
      <td>-98.277778</td>
      <td>0.666667</td>
      <td>0.944444</td>
      <td>7.500000</td>
      <td>82.055556</td>
      <td>0.569031</td>
      <td>0.722222</td>
      <td>18</td>
    </tr>
    <tr>
      <th>S04 Abbedagge</th>
      <td>0.285159</td>
      <td>0.706940</td>
      <td>0.277778</td>
      <td>3.611111</td>
      <td>3.500000</td>
      <td>2.111111</td>
      <td>4.277778</td>
      <td>40.833333</td>
      <td>1.289298</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>S04 Dreams</th>
      <td>0.081275</td>
      <td>0.671343</td>
      <td>0.266667</td>
      <td>-36.800000</td>
      <td>0.866667</td>
      <td>2.400000</td>
      <td>7.400000</td>
      <td>84.800000</td>
      <td>0.678752</td>
      <td>0.533333</td>
      <td>15</td>
    </tr>
    <tr>
      <th>S04 Gilius</th>
      <td>0.137355</td>
      <td>0.736826</td>
      <td>5.833333</td>
      <td>442.083333</td>
      <td>4.000000</td>
      <td>2.083333</td>
      <td>6.250000</td>
      <td>58.000000</td>
      <td>0.814762</td>
      <td>0.666667</td>
      <td>12</td>
    </tr>
    <tr>
      <th>S04 Neon</th>
      <td>0.293243</td>
      <td>0.727295</td>
      <td>-1.600000</td>
      <td>-83.800000</td>
      <td>2.600000</td>
      <td>2.133333</td>
      <td>6.266667</td>
      <td>39.266667</td>
      <td>1.353045</td>
      <td>0.533333</td>
      <td>15</td>
    </tr>
    <tr>
      <th>S04 Odoamne</th>
      <td>0.204055</td>
      <td>0.665810</td>
      <td>4.444444</td>
      <td>-25.500000</td>
      <td>1.500000</td>
      <td>2.555556</td>
      <td>6.111111</td>
      <td>33.000000</td>
      <td>1.016403</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK Crownshot</th>
      <td>0.305696</td>
      <td>0.682608</td>
      <td>11.888889</td>
      <td>264.555556</td>
      <td>3.555556</td>
      <td>1.833333</td>
      <td>4.833333</td>
      <td>40.111111</td>
      <td>1.248497</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK Jenax</th>
      <td>0.231644</td>
      <td>0.560635</td>
      <td>-5.222222</td>
      <td>-460.388889</td>
      <td>2.611111</td>
      <td>2.611111</td>
      <td>4.444444</td>
      <td>32.277778</td>
      <td>1.120304</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK LIMIT</th>
      <td>0.089615</td>
      <td>0.711859</td>
      <td>-8.055556</td>
      <td>-138.944444</td>
      <td>0.888889</td>
      <td>2.500000</td>
      <td>7.833333</td>
      <td>95.555556</td>
      <td>0.699940</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK Trick</th>
      <td>0.114979</td>
      <td>0.642871</td>
      <td>0.222222</td>
      <td>-151.333333</td>
      <td>1.666667</td>
      <td>2.833333</td>
      <td>6.333333</td>
      <td>40.666667</td>
      <td>0.685385</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>SK ZaZee</th>
      <td>0.258067</td>
      <td>0.749928</td>
      <td>-4.166667</td>
      <td>-79.666667</td>
      <td>3.333333</td>
      <td>2.166667</td>
      <td>5.666667</td>
      <td>33.500000</td>
      <td>1.087049</td>
      <td>0.500000</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Cabochard</th>
      <td>0.195603</td>
      <td>0.555988</td>
      <td>-6.777778</td>
      <td>-393.833333</td>
      <td>1.944444</td>
      <td>2.277778</td>
      <td>4.000000</td>
      <td>36.111111</td>
      <td>0.942578</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Comp</th>
      <td>0.337604</td>
      <td>0.788173</td>
      <td>2.222222</td>
      <td>94.888889</td>
      <td>4.000000</td>
      <td>2.277778</td>
      <td>4.555556</td>
      <td>45.722222</td>
      <td>1.281572</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Labrov</th>
      <td>0.073434</td>
      <td>0.792618</td>
      <td>0.611111</td>
      <td>-145.833333</td>
      <td>0.388889</td>
      <td>2.611111</td>
      <td>8.222222</td>
      <td>103.111111</td>
      <td>0.552762</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>VIT Milica</th>
      <td>0.290508</td>
      <td>0.692663</td>
      <td>-6.166667</td>
      <td>-254.000000</td>
      <td>3.111111</td>
      <td>1.888889</td>
      <td>4.388889</td>
      <td>45.000000</td>
      <td>1.123236</td>
      <td>0.388889</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Caedrel</th>
      <td>0.104300</td>
      <td>0.608204</td>
      <td>0.333333</td>
      <td>-85.388889</td>
      <td>1.444444</td>
      <td>2.222222</td>
      <td>4.055556</td>
      <td>52.111111</td>
      <td>0.576975</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Kryze</th>
      <td>0.241151</td>
      <td>0.546163</td>
      <td>-0.722222</td>
      <td>134.000000</td>
      <td>1.888889</td>
      <td>2.444444</td>
      <td>2.944444</td>
      <td>30.111111</td>
      <td>0.992337</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Patrik</th>
      <td>0.332169</td>
      <td>0.704439</td>
      <td>8.944444</td>
      <td>250.666667</td>
      <td>3.611111</td>
      <td>2.055556</td>
      <td>3.500000</td>
      <td>42.000000</td>
      <td>1.320411</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Special</th>
      <td>0.233506</td>
      <td>0.608768</td>
      <td>-3.666667</td>
      <td>-117.888889</td>
      <td>2.000000</td>
      <td>2.500000</td>
      <td>3.833333</td>
      <td>35.722222</td>
      <td>0.950198</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
    <tr>
      <th>XL Tore</th>
      <td>0.088873</td>
      <td>0.677933</td>
      <td>-6.166667</td>
      <td>-88.888889</td>
      <td>0.166667</td>
      <td>2.388889</td>
      <td>6.388889</td>
      <td>92.500000</td>
      <td>0.662340</td>
      <td>0.444444</td>
      <td>18</td>
    </tr>
  </tbody>
</table>
</div>



You can also target a specific player and get stats on the champion he played


```python
df[df["name"] == "G2 Caps"].groupby("champion").agg(
    played=("name","count"), 
    kill_participation=("kill_participation","mean"), 
    cs_diff_at_15=("cs_diff_at_15","mean"), 
    ga_at_15=("ga_at_15","mean"), 
    winrate=("win","mean")
)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>played</th>
      <th>kill_participation</th>
      <th>cs_diff_at_15</th>
      <th>ga_at_15</th>
      <th>winrate</th>
    </tr>
    <tr>
      <th>champion</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Azir</th>
      <td>1</td>
      <td>0.833333</td>
      <td>13.0</td>
      <td>955.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>Cassiopeia</th>
      <td>1</td>
      <td>0.714286</td>
      <td>37.0</td>
      <td>128.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>Ekko</th>
      <td>1</td>
      <td>0.750000</td>
      <td>-2.0</td>
      <td>38.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>Galio</th>
      <td>1</td>
      <td>1.000000</td>
      <td>-18.0</td>
      <td>-1312.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>Kassadin</th>
      <td>1</td>
      <td>0.200000</td>
      <td>-4.0</td>
      <td>-102.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>Kog'Maw</th>
      <td>2</td>
      <td>0.787879</td>
      <td>-3.5</td>
      <td>1119.5</td>
      <td>0.5</td>
    </tr>
    <tr>
      <th>LeBlanc</th>
      <td>1</td>
      <td>0.560000</td>
      <td>-19.0</td>
      <td>-163.0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>Orianna</th>
      <td>1</td>
      <td>0.777778</td>
      <td>6.0</td>
      <td>-307.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>Syndra</th>
      <td>2</td>
      <td>0.663534</td>
      <td>4.5</td>
      <td>428.5</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>Twisted Fate</th>
      <td>2</td>
      <td>0.760417</td>
      <td>16.0</td>
      <td>855.5</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>Zoe</th>
      <td>5</td>
      <td>0.671642</td>
      <td>-1.0</td>
      <td>574.4</td>
      <td>0.8</td>
    </tr>
  </tbody>
</table>
</div>



Now that you have the data and the keys to analyze esports data, make your own stats with more metrics, extend the games analyzed to Playoffs, and even to more regions like LCK and LPL. For other metric, I advise to take a look at this sheet made by LEC analyst that explains the metrics and how to compute them : https://docs.google.com/spreadsheets/d/1hBzgxIPpoBqinOoXCnPvpqUDz4Ja9Lmfm7tqgU3kHRw/edit#gid=1424953934

You may also want to look at last year article : https://hextechlab.com/2019/10/25/worlds2019/ <br /> The first part to gather the data is outdated, but the analysis on champions is still working and compatible with the data gathered earlier in this article.


```python

```
