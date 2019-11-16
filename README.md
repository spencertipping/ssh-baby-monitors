# SSH baby monitors
We don't trust anyone and have some old Thinkpads that run Ubuntu. Rather than
buying a baby monitor that uploads baby-noises to the NSA, we use wifi-connected
Linux boxes to forward baby noises over SSH.

```sh
$ sudo apt install alsa-utils openssh-server      # on the listening machine
$ sudo apt install sox                            # on the playback machine
```


## The basic idea
Run `arecord` to convert sound to data, forward data over SSH, then play it
locally. While we're at it, let's clip out baseline noise using `compand`:

```sh
$ ssh monitormachine 'arecord -f cd -t raw -' \
    | play -e signed-integer -b 16 -L -c 2 -r 44100 -t raw - \
           compand .01,.01 -inf,-40,-inf,-40,-40 0 -90 .1
```

`man play` goes through the options, but basically we're setting up the format
to read what `arecord` calls `-f cd`: 44100Hz stereo 16bit LE signed-int
samples.

- `-e signed-integer`: each sample is a signed integer, as opposed to a float
- `-b 16`: each sample integer has 16 bits
- `-L`: ...encoded little-endian
- `-c 2`: two channels (stereo)
- `-r 44100`: sample rate
- `-t raw`: don't look for any header data


### Why `-t raw` and all the format stuff instead of WAV?
`arecord` cuts off WAV data after 2GiB, which is about three and a half hours (I
think it uses a signed 32-bit length field). Since we don't really care about
the length, we can skip the container format and tell `play` how to decode the
samples. `arecord` will run until you interrupt it; it won't quit on its own.
