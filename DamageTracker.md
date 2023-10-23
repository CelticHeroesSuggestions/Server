# DamageTracker

What: A damage tracker leaderboard for certain mobs and raids.

Why: Many players want more detailed information about how much damage players do at raids, this would be an effective way to track damage for clans.

How: Track the damage server-side, and store in the database after the mob dies. Use a patchserver page to display the leaderboard, and repurpose a UI button to open the patchserver page.

Notes: I have no idea if this will actually work, it's just a proof-of-concept with some educated guesses. Maybe DECA will be able to adapt it to make a functioning leaderboard.

## Database

Make a new table in the `rpgdb`:

`CREATE TABLE damagetracker (mob_name VARCHAR(255), character_name VARCHAR(255), damage INT, death_time INT);`

Set the `mob_name` and `character_name` columns to both be Primary Keys.

## Server

In `Program.cs`, add a new static dictionary for the damage tracking, and an array of mobs to track:

```
public static class Program {
  ...  // class-wide variable initializations

  // DamageTracker storage
  public static Dictionary<string, Dictionary<string,int>> DamageTracker = new Dictionary<string, Dictionary<string,int>>();
  public static string[] DamageTrackerMobs = {"Aggragoth", "Hrungnir", "Mordris", "Efnisien the Necromancer", "Gelebron", "Bloodthorn", "Dhiothu"};

  ...  // other class-wide variable initializations
}
```

In `CombatEntity.cs`, record damage when the mob takes hits and update the database when the mob dies:

```
public CombatDamageMessageData TakeDamage(...) {
  ...  // damage is calculated and damage message type is determined

  // DamageTracker: add to the dictionary if tracking this mob
  if (casterId != m_serverId && dmgMsgType == CombatManager.DamageMessageType.PlayerToMob && Program.DamageTrackerMobs.Contains(Name)) {
      // store the data in memory, and transfer to SQL on mob death (somewhere else)
      if (!Program.DamageTracker.ContainsKey(Name))
         Program.DamageTracker.Add(Name, new Dictionary<string, int>());
      if (!Program.DamageTracker[Name].ContainsKey(caster.Name))
         Program.DamageTracker[Name].Add(caster.Name, 0);
      Program.DamageTracker[Name][caster.Name] += damage;
  }

  ...  // message reply is constructed
}

...

public virtual void Died() {
  ...

  // DamageTracker: update the SQL table
  if (Program.DamageTracker.ContainsKey(Name)) {
      // pull the timestamp
      int death_time = (int)DateTimeOffset.Now.ToUnixTimeSeconds();
      // update the database
      foreach (string character_name in Program.DamageTracker[Name].Keys) {
          var parameters = new List<MySqlParameter>();
          parameters.Add(new MySqlParameter("@mob_name", Name));
          parameters.Add(new MySqlParameter("@character_name", character_name));
          parameters.Add(new MySqlParameter("@damage", Program.DamageTracker[Name][character_name]));
          parameters.Add(new MySqlParameter("@death_time", death_time));
          Program.Processor.WorldDb.RunCommand(
              "DELETE FROM damagetracker WHERE mob_name = @mob_name; INSERT INTO damagetracker (mob_name, character_name, damage, death_time) VALUES (@mob_name, @character_name, @damage, @death_time) ON DUPLICATE KEY UPDATE damage = @damage, death_time = @death_time; ",
              parameters);
      }
      // clear from the damage tracker
      Program.DamageTracker.Remove(Name);
  }
}
```

## Patchserver

In the patchserver, create two new files: `DamageTracker.aspx` and `DamageTracker.aspx.cs`, and update the `Web.config` file.

Code for `DamageTracker.aspx`:
```
<%@ Page Language="C#" AutoEventWireup="true" CodeFile="DamageTracker.aspx.cs" Inherits="Rpg.PatchServer.damagetracker" %>

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" >
<head runat="server">
    <title>Damage Leaderboard</title>
    <link href='https://fonts.googleapis.com/css?family=Lato:400,700' rel='stylesheet' type='text/css'>
</head>

<style>
    body {
        display: flex;
        height: 100vh;
        width: 100vw;
        flex-direction: column;
        align-items: center;
        margin: 0;
        font-family: 'Lato', sans-serif;
        color: #272727;
    }
    div {
        display: flex;
    }
    div.title {
        background-color: #95b8d1;
        width: 100%;
        justify-content: center;
        font-size: 2.5rem;
        padding: 1.5rem 0 1.5rem 0;
        border-bottom: .25rem solid #809bce;
        margin-bottom: 2rem;
        box-shadow: 0 0 .55rem grey;
    }
    div.ranking-container {
        width: 100%;
        flex: 1;
        flex-direction: row;
        justify-content: space-evenly;
        flex-wrap: wrap;
    }
    div.ranking-item {
        height: 30rem;
        width: 20rem;
        background-color: #d6eadf;
        box-shadow: .25rem .25rem .25rem grey;
        padding: 2rem;
        border-radius: .5rem;
        margin: 1rem;
        flex-direction: column;
    }
    div.ranking-title {
        font-size: 2rem;
        justify-content: center;
        margin-bottom: .5rem;
        text-align: center;
    }
    div.ranking-time {
        font-size: 1rem;
        justify-content: center;
        padding-bottom: .5rem;
        margin-bottom: .5rem;
        border-bottom: .12rem solid grey;
    }
    div.ranking-player {
        font-size: 1.25rem;
        flex-direction: row;
        justify-content: space-between;
    }
</style>

<body>
    <div class="title">
        Damage Leaderboard
    </div>
    <div id="rankingContainer" class="ranking-container" runat="server">
    </div>
</body>
</html>
```

Code for `DamageTracker.aspx.cs`:
```
using System;
using System.Configuration;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using MySqlConnector;

namespace Rpg.PatchServer
{
    public partial class damagetracker : System.Web.UI.Page
    {
        protected void Page_Load(object sender, EventArgs e)
        {
            // connect to the database
            MySqlConnection rpgdb = new MySqlConnection(ConfigurationManager.AppSettings["RpgdbConnectionInfo"]);
            rpgdb.Open();
            MySqlCommand cmd = new MySqlCommand("select * from damagetracker", rpgdb);
            MySqlDataReader rdr = cmd.ExecuteReader();

            // populate the damage tracking dictionary
            Dictionary<string, Dictionary<string, Tuple<int,long>>> DamageTracker = new Dictionary<string, Dictionary<string, Tuple<int,long>>>();
            while (rdr.Read())
            {
                string mob_name = (string) rdr[0];
                string character_name = (string) rdr[1];
                int damage = (int) rdr[2];
                long timestamp = (int) rdr[3];
                if (!DamageTracker.ContainsKey(mob_name))
                    DamageTracker.Add(mob_name, new Dictionary<string, Tuple<int,long>>());
                if (!DamageTracker[mob_name].ContainsKey(character_name))
                    DamageTracker[mob_name].Add(character_name, new Tuple<int,long>(damage, timestamp));
            }
            rdr.Close();
            rpgdb.Close();

            // display the rankings
            foreach (string mob_name in DamageTracker.Keys) {
                // create the panel
                System.Web.UI.HtmlControls.HtmlGenericControl rankingPanel = 
                    new System.Web.UI.HtmlControls.HtmlGenericControl("DIV");
                rankingPanel.ID = "ranking-" + mob_name;
                rankingPanel.Attributes.Add("class", "ranking-item");
                rankingContainer.Controls.Add(rankingPanel);

                // fill in the panel
                
                // title
                System.Web.UI.HtmlControls.HtmlGenericControl rankingTitle = 
                    new System.Web.UI.HtmlControls.HtmlGenericControl("DIV");
                rankingTitle.ID = "ranking-title-" + mob_name;
                rankingTitle.Attributes.Add("class", "ranking-title");
                rankingPanel.Controls.Add(rankingTitle);
                rankingTitle.InnerText = mob_name;

                // time
                System.Web.UI.HtmlControls.HtmlGenericControl rankingTime = 
                    new System.Web.UI.HtmlControls.HtmlGenericControl("DIV");
                rankingTime.ID = "ranking-time-" + mob_name;
                rankingTime.Attributes.Add("class", "ranking-time");
                rankingPanel.Controls.Add(rankingTime);

                // players
                int rank = 1;
                var sortedDamageTracker = DamageTracker[mob_name].OrderByDescending(x => x.Value);
                foreach (var damageTracked in sortedDamageTracker) {
                    // break after 15 players
                    if (rank > 15) {
                        break;
                    }
                    // container
                    System.Web.UI.HtmlControls.HtmlGenericControl rankingPlayer = 
                        new System.Web.UI.HtmlControls.HtmlGenericControl("DIV");
                    rankingPlayer.Attributes.Add("class", "ranking-player");
                    rankingPlayer.ID = "ranking-player-" + mob_name;
                    rankingPanel.Controls.Add(rankingPlayer);

                    // name
                    System.Web.UI.HtmlControls.HtmlGenericControl rankingPlayerName = 
                        new System.Web.UI.HtmlControls.HtmlGenericControl("DIV");
                    rankingPlayerName.ID = "ranking-player-name-" + mob_name + "-" + damageTracked.Key;
                    rankingPlayer.Controls.Add(rankingPlayerName);
                    rankingPlayerName.InnerHtml = "#" + rank + "&emsp;" + damageTracked.Key;

                    // damage
                    System.Web.UI.HtmlControls.HtmlGenericControl rankingPlayerDamage = 
                        new System.Web.UI.HtmlControls.HtmlGenericControl("DIV");
                    rankingPlayerDamage.ID = "ranking-player-damage" + mob_name + "-" + damageTracked.Key;
                    rankingPlayer.Controls.Add(rankingPlayerDamage);
                    rankingPlayerDamage.InnerText = damageTracked.Value.Item1.ToString();

                    // death time if not set
                    if (rankingTime.InnerText == "") {
                        DateTime dateTime = DateTimeOffset.FromUnixTimeSeconds(damageTracked.Value.Item2).LocalDateTime;
                        rankingTime.InnerText = dateTime.ToString("dddd MMMM dd 'at' HH:mmtt");
                    }

                    rank += 1;
                }
            }
        }
    }
}
```

No other patchserver routes (that I know of) use the world database (`rpgdb`), so update the patchserver's `Web.config` file.

Replace `<compilation targetFramework="4.6">` with:
```
<compilation targetFramework="4.6">
  <assemblies>
    <add assembly="System.Runtime, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
  </assemblies>
</compilation>
```

and add the `rpgdb` access parameters under `<appSettings>`, alongside the other SQL connections:

`<add key="RpgdbConnectionInfo" value="SERVER=localhost;DATABASE=ch_rpgdb;UID=(YOUR SQL USERNAME);PASSWORD=(YOUR SQL PASSWORD)" />`

## Access

And that's it! After refreshing the IIS server the damage leaderboard should be accessible:

`https://(patchserverurl)/patchserver/damagetracker.aspx`

To change the mobs being tracked, edit the `DamageTrackerMobs` variable in `Program.cs`
