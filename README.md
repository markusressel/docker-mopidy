This is a fork of [wernight/docker-mopidy](https://github.com/wernight/docker-mopidy) that includes a snapserver inside the same container to more easily be able to create a multi-room media player system.

# ORIGINAL README

# Features
  * Debian Jessy base image
  * Includes backend extensions for:
    * [Mopidy-Spotify](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-spotify) for **[Spotify](https://www.spotify.com/us/)** (Premium)
    * [Mopidy-GMusic](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-gmusic) for **[Google Play Music](https://play.google.com/music/listen)**
    * [Mopidy-SoundClound](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-soundcloud) for **[SoundCloud](https://soundcloud.com/stream)**
    * [Mopidy-Pandora](https://github.com/rectalogic/mopidy-pandora) for **[Pandora](https://www.pandora.com/)**
    * [Mopidy-YouTube](https://docs.mopidy.com/en/latest/ext/backends/#mopidy-youtube) for **[YouTube](https://www.youtube.com)**
    * [Mopidy-Local]https://github.com/mopidy/mopidy-local) album art image serving
  * Includes frontend extensions:
    * [Iris](https://github.com/jaedb/Iris) web extension
    * [Mopidy-Moped](https://mopidy.com/ext/moped/) web extension
  
  * Can run as any user and runs as UID/GID `84044` user inside the container by default (for security reasons).
  * Includes [Snapcast](https://github.com/badaix/snapcast) server

TODO: how would one achieve this?
You may install additional [backend extensions](https://docs.mopidy.com/en/latest/ext/backends/).

# Usage

## Playing sound from the container
There are various ways to have the audio from Mopidy running in your container to play on your system's audio output. Have a look at [wernight/docker-mopidy](https://github.com/wernight/docker-mopidy#playing-sound-from-the-container) for a selection of options.

### General usage

    $ docker run -d \
        $PUT_HERE_EXRA_DOCKER_ARGUMENTS_FOR_AUDIO_TO_WORK \
        -v "$PWD/media:/var/lib/mopidy/media:ro" \
        -v "$PWD/local:/var/lib/mopidy/local" \
        -p 6600:6600 -p 6680:6680 \
        --user $UID:$GID \
        wernight/mopidy \
        mopidy \
        -o spotify/username=USERNAME -o spotify/password=PASSWORD \
        -o gmusic/username=USERNAME -o gmusic/password=PASSWORD \
        -o soundcloud/auth_token=TOKEN

Most arguments are optional (see some examples below):

  * Docker arguments:
      * `$PUT_HERE_EXRA_DOCKER_ARGUMENTS_FOR_AUDIO_TO_WORK` should be replaced
        with some arguments that work to play audio from within the docker
        container as tested above.
      * `-v ...:/var/lib/mopidy/media:ro` - (optional) Path to directory with local media files.
      * `-v ...:/var/lib/mopidy/local` - (optional) Path to directory to store local metadata such as libraries and playlists in.
      * `-p 6600:6600` - (optional) Exposes MPD server (if you use for example ncmpcpp client).
      * `-p 6680:6680` - (optional) Exposes HTTP server (if you use your browser as client).
      * `-p 5555:5555/udp` - (optional) Exposes [UDP streaming for FIFE sink](https://github.com/mopidy/mopidy/issues/775) (e.g. for visualizers).
      * `--user $UID:$GID` - (optional) You may run as any UID/GID, and by default it'll run as UID/GID `84044` (`mopidy:audio` from within the container).
        The main restriction is if you want to read local media files: That the user (UID) you run as should have read access to these files.
        Similar for other mounts. If you have issues, try first as `--user root`.
  * Mopidy arguments (see [mopidy's command](https://docs.mopidy.com/en/latest/command/) for possible additional options),
    replace `USERNAME`, `PASSWORD`, `TOKEN` accordingly if needed, or disable services (e.g., `-o spotify/enabled=false`):
      * For *Spotify* you'll need a *Premium* account.
      * For *Google Music* use your Google account (if you have *2-Step Authentication*, generate an [app specific password](https://security.google.com/settings/security/apppasswords)).
      * For *SoundCloud*, just [get a token](https://www.mopidy.com/authenticate/) after registering.

NOTE: Any user on your system may run `ps aux` and see the command-line you're running, so your passwords may be exposed.
A safer option if it's a concern, is using putting these passwords in a Mopidy configuration file based on [mopidy.conf](mopidy.conf):

    [core]
    data_dir = /var/lib/mopidy

    [local]
    media_dir = /var/lib/mopidy/media

    [audio]
    output = tee name=t ! queue ! autoaudiosink t. ! queue ! udpsink host=0.0.0.0 port=5555

    [m3u]
    playlists_dir = /var/lib/mopidy/playlists

    [http]
    hostname = 0.0.0.0

    [mpd]
    hostname = 0.0.0.0

    [spotify]
    username=USERNAME
    password=PASSWORD

    [gmusic]
    username=USERNAME
    password=PASSWORD

    [soundcloud]
    auth_token=TOKEN

Then run it:

    $ docker run -d \
        $PUT_HERE_EXRA_DOCKER_ARGUMENTS_FOR_AUDIO_TO_WORK \
        -v "$PWD/media:/var/lib/mopidy/media:ro" \
        -v "$PWD/local:/var/lib/mopidy/local" \
        -v "$PWD/mopidy.conf:/config/mopidy.conf" \
        -p 6600:6600 -p 6680:6680 \
        --user $UID:$GID \
        wernight/mopidy


##### Example using HTTP client to stream local files

 1. Give read access to your audio files to user **84044**, group **84044**, or all users (e.g., `$ chgrp -R 84044 $PWD/media && chmod -R g+rX $PWD/media`).
 2. Index local files:

        $ docker run --rm \
            --device /dev/snd \
            -v "$PWD/media:/var/lib/mopidy/media:ro" \
            -v "$PWD/local:/var/lib/mopidy/local" \
            -p 6680:6680 \
            wernight/mopidy mopidy local scan

 3. Start the server:

        $ docker run -d \
            -e "PULSE_SERVER=tcp:$(hostname -i):4713" \
            -e "PULSE_COOKIE_DATA=$(pax11publish -d | grep --color=never -Po '(?<=^Cookie: ).*')" \
            -v "$PWD/media:/var/lib/mopidy/media:ro" \
            -v "$PWD/local:/var/lib/mopidy/local" \
            -p 6680:6680 \
            wernight/mopidy

 4. Browse to http://localhost:6680/

#### Example using [ncmpcpp](https://docs.mopidy.com/en/latest/clients/mpd/#ncmpcpp) MPD console client

    $ docker run --name mopidy -d \
        -v /run/user/$UID/pulse:/run/user/105/pulse \
        wernight/mopidy
    $ docker run --rm -it --net container:mopidy wernight/ncmpcpp ncmpcpp

Alternatively if you don't need visualizers you can do:

    $ docker run --rm -it --link mopidy:mopidy wernight/ncmpcpp ncmpcpp --host mopidy


### Feedbacks

Having more issues? [Report a bug on GitHub](https://github.com/wernight/docker-mopidy/issues). Also if you need some additional extensions/plugins that aren't already installed (please explain why).


### Alsa Audio

For non debian distros. The gid for audio group in /etc/group must be 29 to match debians default as `audio:x:29:<your user outside of docker>` this is to match the user id inside the docker container. You'll also need to add the `output = alsasink` config line under the audio section in your `mopidy.conf`.

```
$ docker run -d -rm \
  --device /dev/snd \
  --name mopidy \
  --ipc=host \
  --privileged \
  -v $HOME/.config/mopidy:/var/lib/mopidy/.config/mopidy/ \
  -p 6600:6600/tcp -p 6680:6680/tcp -p 5555:5555/udp \
  mopidy
```

