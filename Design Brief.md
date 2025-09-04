# Program Layout
The program will be broken into various "handlers" to handle various aspects of the program. These would include but are not limited to:

1) GuiHandler - handles drawing onto the screen.
2) AudioHandler - handles music playback.
3) ConfigHandler - Handles any reading and writing to the config file.
4) InputHandler - Handles user input via the rotary encoders.

There will also be a main.ino that acts as the backbone of the program.

The operation of the program will be built around the "Observer" design pattern. Handlers will "Publish" updates and "Subscribers" will listen and execute functions based on the event.

The operation of the inputs  will be state-based. When the screen is off the Inputs will cause different behaviour to when the screen is on. 

**Note that all code examples are pseudo-code**
### Example
```c
// ------------------------------------------------------------
// ENUM: EventType
// Represents the types of events the system can broadcast.
// In this example, only DOUBLE_TAP is used to signal a track skip.
// ------------------------------------------------------------
enum EventType { DOUBLE_TAP };


// ------------------------------------------------------------
// INTERFACE: Listener
// Purpose: Defines the contract for any class that wants to
// "listen" for events. Each subscriber must implement onEvent().
//
// Method:
//    onEvent(EventType e)
//    Input: e - The type of event that occurred.
//    Output: None (subscriber decides what to do).
// ------------------------------------------------------------
interface Listener {
    void onEvent(EventType e);
}


// ------------------------------------------------------------
// CLASS: InputHandler (Publisher)
// Purpose: Detects user input (button taps) and broadcasts events
// to all registered subscribers using the Observer pattern.
//
// Key Methods:
//    add(Listener* l)
//       Input: Pointer to a Listener object.
//       Action: Registers this object to receive events.
//
//    checkInput()
//       Purpose: Called repeatedly (e.g., in loop) to check for button input.
//       Detects double taps by timing two presses within THRESHOLD.
//       If detected, calls notify().
//
//    notify(EventType e)
//       Purpose: Sends an event to all subscribers.
//       Input: e - The event to broadcast.
// ------------------------------------------------------------
class InputHandler {
    List<Listener*> subs;        // List of subscribers
    unsigned long lastTap = 0;   // Time of last button press
    const unsigned long THRESHOLD = 300; // Max gap (ms) for double tap

    void add(Listener* l) { subs.add(l); }

    void checkInput() {
        if (buttonPressed()) {                  // INPUT: physical button state
            unsigned long now = millis();       // Get current time
            if ((now - lastTap) < THRESHOLD)    // Check if double tap occurred
                notify(DOUBLE_TAP);             // Broadcast DOUBLE_TAP event
            lastTap = now;                      // Update last tap time
        }
    }

    void notify(EventType e) {
        for (auto l : subs) l->onEvent(e);      // OUTPUT: Notify all listeners
    }
}


// ------------------------------------------------------------
// CLASS: AudioHandler (Subscriber)
// Purpose: Responds to events by controlling audio playback.
// Implements the Listener interface.
// Behavior:
//    On DOUBLE_TAP → skips to next track.
// ------------------------------------------------------------
class AudioHandler implements Listener {
    void onEvent(EventType e) {
        if (e == DOUBLE_TAP) skipTrack();       // React to event
    }

    void skipTrack() {
        print("Next track");                    // Action: Advance playlist
    }
}


// ------------------------------------------------------------
// CLASS: GUIHandler (Subscriber)
// Purpose: Updates the display when tracks change.
// Implements the Listener interface.
// Behavior:
//    On DOUBLE_TAP → updates GUI to show next track.
// ------------------------------------------------------------
class GUIHandler implements Listener {
    void onEvent(EventType e) {
        if (e == DOUBLE_TAP) showNext();        // React to event
    }

    void showNext() {
        print("Display next track");            // Action: Update UI
    }
}

```
# Code Flavour
Prioritising readability and explicit code over efficiency and conciseness will ensure maintainability and scalability. Loose coupling is vital for modularity. 
## Comments
All functions and variables should be proceeded by a comment that describes the function of the function etc.
### eg.
```c
//============================================================
// FUNCTION NAME: adjustVolume
//
// PURPOSE:
//   Adjusts the playback volume of the music player to a new
//   level specified by the user. This ensures the value is 
//   within the valid range supported by the hardware.
//
// INPUT PARAMETERS:
//   - newVolume (Integer):
//       Desired volume level from the user. Expected range
//       is 0 to 100, where:
//          0   = completely silent
//          100 = maximum volume
//
// OUTPUT:
//   - (Boolean):
//       Returns TRUE if the volume was successfully set,
//       or FALSE if the input was invalid (e.g., out of range).
//
// BEHAVIOR:
//   1. Check if the requested volume is within 0–100.
//   2. If not valid, do nothing and return FALSE.
//   3. If valid, map the percentage to the hardware scale 
//      (for example, 0–30 steps internally).
//   4. Send the volume command to the audio module.
//   5. Print a debug message showing the new volume.
//   6. Return TRUE to indicate success.
//
// EXAMPLE USAGE:
//   Input:  75
//   Output: TRUE (and sets volume to 75%)
//============================================================
FUNCTION adjustVolume(newVolume):

    // STEP 1: Validate input range
    IF newVolume < 0 OR newVolume > 100:
        PRINT "Error: Volume out of range."
        RETURN FALSE

    // STEP 2: Convert to hardware scale (example: 0-30)
    hardwareVolume ← MAP(newVolume, 0, 100, 0, 30)

    // STEP 3: Send command to audio module
    SEND_COMMAND_TO_AUDIO("SET_VOLUME", hardwareVolume)

    // STEP 4: Debug log
    PRINT "Volume set to " + newVolume + "%"

    // STEP 5: Return success
    RETURN TRUE
END FUNCTION

```
# Hardware
The priority for hardware was voltage, every component can operate on 3.3 volts. This eliminates the need for voltage boosters for 5v components. 

The parts are as follows - sku numbers are for core electronics (C) or Jaycar (J)
1) Brain =  CE09705 (C)
2) Screen = core - WS-13892 (C)
3) SD Card Reader = ADA4682 (C)
4) Rotary Encoders = XC3736 (J)
5) Battery = CE04377 (C) 


# Config

The behaviour and GUI will be based on variables in a settings_config.yaml file. The colour scheme, input behaviour and states will be stored in and read from this file. the file will be stored on the sd card and read during boot to set program variables and written to when users change their settings in the settings menu. this will increase customisability and maintain persistence between boots.

```c 
#============================================================
# SETTINGS CONFIGURATION
# Stores general user preferences and GUI behaviour.
#============================================================

screen:
  timeout_seconds: 30        # Time before display sleeps
  brightness: 80             # Screen brightness (0–100)
  theme: "dark"              # Options: dark, light, custom

inputs:
  rotary_encoder_direction: "clockwise-increase"
  double_tap_threshold_ms: 300
  long_press_threshold_ms: 800

audio:
  default_volume: 75         # Percentage (0–100)
  max_volume: 100
  startup_action: "resume"   # Options: resume, stop
  
storage:
  delete_playes: "False"     # True deletes podcasts after playback stops. 

```
# Menu Layout

**!!!ROUGH LAYOUT!!!**


```c
BASE_MENU
 ├── Now Playing
 │    ├── Music → NOW_PLAYING_MUSIC screen
 │    │     └── Modified Submenu
 │    │           ├── Play/Pause
 │    │           ├── Skip Track Forward
 │    │           ├── Skip Track Backward
 │    │           ├── View Queue
 │    │           │     └── Queue Submenu
 │    │           │           ├── Remove from Queue (per track)
 │    │           │           └── Clear Queue
 │    │           ├── Go to Artist
 │    │           └── Delete → CONFIRM_DELETE
 │    │
 │    └── Podcast → NOW_PLAYING_PODCAST screen
 │          └── Modified Submenu
 │                ├── Play/Pause
 │                ├── Skip Track Forward
 │                ├── Skip Track Backward
 │                ├── Skip 10s Forward
 │                ├── Skip 10s Backward
 │                ├── View Queue
 │                │     └── Queue Submenu (remove/clear)
 │                ├── Mark as Played
 │                ├── Mark as Unplayed
 │                └── Delete → CONFIRM_DELETE
 │
 ├── Music
 │    ├── All Songs → MUSIC_BROWSER (ALL_SONGS)
 │    │     └── Standard Submenu
 │    │           ├── Add to Queue
 │    │           ├── Add to Playlist
 │    │           ├── Go to Artist
 │    │           └── Delete → CONFIRM_DELETE
 │    │
 │    ├── Playlists → MUSIC_BROWSER (PLAYLISTS)
 │    │     └── Standard Submenu (as above)
 │    │
 │    ├── Albums → MUSIC_BROWSER (ALBUMS)
 │    │     └── Standard Submenu (as above)
 │    │
 │    └── Artists → MUSIC_BROWSER (ARTISTS)
 │          └── Standard Submenu (as above)
 │
 ├── Podcasts
 │    ├── All Podcasts → PODCAST_BROWSER (ALL_EPISODES)
 │    │     ├── Sort By (release date/date added)
 │    │     └── Standard Submenu
 │    │           ├── Add to Queue
 │    │           ├── Add to Playlist
 │    │           ├── Go to Artist
 │    │           ├── Delete → CONFIRM_DELETE
 │    │           ├── Mark as Played
 │    │           └── Mark as Unplayed
 │    │
 │    └── Podcast (Specific Artist) → PODCAST_BROWSER (BY_ARTIST)
 │          └── Standard Submenu (as above)
 │
 └── Settings
      ├── Screen Timeout
      ├── Input Behaviour
      ├── Sleep Timer
      └── Shutdown Menu
            ├── Cancel
            └── Shutdown (stop + deep sleep)
```

# Files
During runtime the device would have several files on it. 
The config files are stored on the sd and get read from during runtime. should the files not exist during boot, the player will generate them with a default config layout.
```c
├── media
│   ├── music
│   │   ├── track1 - Sunrise Drive.mp3
│   │   ├── track2 - Electric Horizon.mp3
│   │   ├── track3 - Midnight Groove.mp3
│   │   ├── track4 - Distant Echoes.mp3
│   │   └── track5 - Ocean Breeze.mp3
│   │
│   └── podcasts
│       ├── ep01 - The Future of Tech.mp3
│       ├── ep02 - History Uncovered.mp3
│       ├── ep03 - Science Weekly.mp3
│       ├── ep04 - Creative Minds.mp3
│       └── ep05 - World News Roundup.mp3
│
└── .config
    ├── settings_config.yaml
    └── playback_config.yaml
```

# Podcast
The core motive for this project is dynamic controls based on the audio being played. When a podcast is played the "Now Playing" screen will include "skip 10s" buttons that will replace the "Skip Track". An additional feature is podcast bookmarking. This feature would involve writing the current listening time to the playback_config.yaml file. The write would happen every 10 seconds, whenever the track gets paused and whenever another track gets played. this ensures that the track picks up where it was left off the next time it is played. 

Additionally a "Played" flag would be tied to each podcast track. When the track has < 5 seconds of runtime left (adjustable in settings), the flag would be set to "True". This would allow the desktop application or even the player itself to remove played podcasts from the device and free up storage space. 
The flags are stored in a tracklist in the playback_config.yaml file. This file would index all of the media on the card and mark them with flags where appropriate. 

```c
#============================================================
# PLAYBACK CONFIGURATION
# Tracks playback state, podcast bookmarks, and played flags.
#============================================================

current_track:
  path: "media/music/track2 - Electric Horizon.mp3"
  position_seconds: 42       # Resume from 42s if resumed

podcasts:
  - file: "media/podcasts/ep01 - The Future of Tech.mp3"
    position_seconds: 180    # Resume from 3 minutes
    played: false
  - file: "media/podcasts/ep02 - History Uncovered.mp3"
    position_seconds: 0
    played: true
  - file: "media/podcasts/ep03 - Science Weekly.mp3"
    position_seconds: 945
    played: false

```


# Playlists
The playback_config.yaml file also keeps track of playlists. Rather than copying the files into different playlist folders, the playback_config.yaml will have stored what playlists each track is part of and the program will reference the list when displaying playlists. 
```c
#============================================================
# PLAYLIST DEFINITIONS
# Each playlist contains references to media file paths.
#============================================================

playlists:
  Chill Vibes:
    - "media/music/track1 - Sunrise Drive.mp3"
    - "media/music/track4 - Distant Echoes.mp3"
    - "media/music/track5 - Ocean Breeze.mp3"

  Workout:
    - "media/music/track2 - Electric Horizon.mp3"
    - "media/music/track3 - Midnight Groove.mp3"

  Podcasts To Finish:
    - "media/podcasts/ep01 - The Future of Tech.mp3"
    - "media/podcasts/ep03 - Science Weekly.mp3"

```