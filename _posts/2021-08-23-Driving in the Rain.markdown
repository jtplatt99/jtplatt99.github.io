---
layout: post
title:  "Driving in the Rain"
date:   2021-08-23 12:00:00 -0500
categories: development
tags: openstreetmap rainviewer termux
---

I think we've all been in this situation before: you're out for a drive and out of nowhere it starts raining. Well, less so rain and more so torrential downpour. I do tend to be a little risk averse, so maybe this is just me, but I think there comes a point where even the most intrepid drivers need to pull into the shoulder and wait it out. I've certainly done so plenty of times, but I often wonder if I'd be passed the worst of it if I just kept driving another minute or two.

Thankfully, the pieces slowly fell together this summer and I've found a way to always know.

{:refdef: style="text-align: center; display: block; margin: 0 auto;"}
![Screenshot from Open Street Map for Android](/static/OsmAndHikingTrip.jpg){: width="200"}
{: refdef}
My latest hiking trip in the Dolly Sods Wilderness plotted in OsmAnd+.<br><br>
{: refdef}

It all starts with [Open Street Map](https://www.openstreetmap.org), specifically the [OsmAnd+](https://f-droid.org/en/packages/net.osmand.plus/) android app. I actually got into this app because it's great for hiking since you can easily download maps for offline use, including hiking and topographical maps, but you can also use it for regular navigation while driving, just like Google Maps. Anyway, one of its other features is the ability to add 'Overlay' and 'Underlay' maps. After one of those dreaded drives through a downpour, I got to thinking: Wouldn't it be great if I could just see the weather radar right on the map while I was driving? That way I could see how big the storm is and decide if I need to pull over or not.

As per usual, I am not the first person to think of this. The right google search will lead to [this post by tweakradje](https://sites.google.com/site/tweakradje/android/osmand-navigation) discussing how they use OsmAnd while biking. They found that while they were unable to use the [Rainviewer API](https://www.rainviewer.com/api.html), they found success using the [OpenWeatherMap API](https://openweathermap.org/api/weathermaps).

# Early Attempts

{:refdef: style="text-align: center; display: block; margin: 0 auto;"}
![Screenshot from Open Street Map with Open Weather Map Precipitation Overlay](/static/OsmAndOWM.jpg){: width="200"}
{: refdef}
The Open Weather Map overlay in OsmAnd+.<br><br>
{: refdef}

I also was able to get the Open Weather Map overlay working in Open Street Map, but I couldn't help but notice that the overlay only displays in blue. I looked at the API to see if there was away to change the color scheme, but this doesn't seem possible. One option could be looking at the [Weather Maps 2.0 API](https://openweathermap.org/api/weather-map-2), but unfortunately this requires a [developer subscription](https://openweathermap.org/price)...and I definitely wasn't planning on paying for that!

I wasn't happy with the way the overlay worked, but I hit a roadblock, plus my internship was just starting to pick up so I decided to shelf the project for the time being. It sat more or less untouched until about a week ago when I was driving back down to Alabama for the start of the fall semester, where, you guessed it, we hit a lot of rain.

The issue is that while RainViewer has very up to date, free, and customizable API for accessing its weather radar tiles, the API requires that the client provide a timestamp for the tile data that it wants to access. Open Street Map simply doesn't have the ability to specify the time stamp when making a request, even though the API is otherwise extremely straight forward.

# The Solution

On the drive, I realized that I didn't have to directly query the RainViewer API from Open Street Map, I could use a proxy server. The proxy server would only require the location data for the desired tile, then itself would determine the required time stamp and retreive the data from RainViewer, before passing it back to Open Street Map. The problem is, do I really want to open a server up to the internet? Even if it can only make request to RainViewer, its not something I was willing to do. Plus, why waste another computer's resources when I can just run the proxy on my phone itself?

#### Proxy Server on a Phone?

Yes, you heard me right, you can run a web server on a phone! For some of you this probably isn't all that surprising, but I expect that the thought never really occured to most people. Anyway, there are plenty of ways to run a web server of a phone, but since I want to run a proxy that modifies the API calls before the go out to RainViewer, I figured the best way would be with python. A simple google search gives dozens of ways to implement a proxy server, I decided to start with [this one](https://levelup.gitconnected.com/how-to-build-a-super-simple-http-proxy-in-python-in-just-17-lines-of-code-a1a09192be00) which looked to be one of the simplest.

The way retrieving radar image tiles from RainViewer works is that first the [weather-maps API](https://api.rainviewer.com/public/weather-maps.json) must be queried to get a list of currently available weather radar data. From there, the path and timestamp to access the most recent data can be obtained from the last entry in the radar.past array. Using the path and timestamp, plus the tile location data from the Open Street Map request, a URL can be constructed to retrieve the actual desired radar tile.

As for the changes, I updated the `do_GET` function to query the RainViewer API to get the most up to date time stamps, and construct the URL for retrieving the radar tiles from RainViewer. Elsewhere I had to update `SocketServer`, `SimpleHTTPServer`, and `urllib` to `socketserver`, `http.server`, and `urllib.request` respectively due to changes in the standard libraries between Python2 and Python3. I also updated the `socketserver.ForkingTCPServer()` to only accept requests from the localhost.

After an hour or two of development (which I did on my phone in Termux), the proxy server worked!

{:refdef: style="text-align: center; display: block; margin: 0 auto;"}
![Screenshot from Open Street Map with RainViewer Precipitation Overlay](/static/OsmAndRainViewer.jpg){: width="200"}
![Screenshot from Termux Running the Proxy Server](/static/TermuxProxyServer.jpg){: width="200"}
{: refdef}
The RainViewer overlay in OsmAnd+ on the left and the proxy server running in Termux on the right.<br><br>
{: refdef}

Now, I can honestly say that having the weather radar displayed right on the map while navigating is amazing. I had the weather overlay up and running for the last two hours of the drive down, and even in that short time it was really useful. I could easily see when rain was coming up, and, even better, I could see right how much more rain was ahead. I was surprised how accurate RainViewer is; all the data seemed spot on, even showing faint blue in sections where it was drizzling. I wonder how long it will be until Google gets around to implementing something similar in Google Maps?

Panning around on the map after arriving at my apartment, I did notice that I ran into a lot more performance issues than when driving. I think this was down to my early implementation for getting the current timestamp that queried the RainViewer weather maps API everytime a map tile needed to be retrieved. As a simple fix, I use the `requests_cache` library to install an HTTP cache to cache the API response for 5 minutes. I think this is reasonable considering that the response is only expected to update once every 10 minutes.

# The Code

My Python3 code is shown below. If you plan on resusing it, keep in mind it has dependencies on the `requests` and `requests_cache` libraries.

{% highlight python %}
import requests
import requests_cache
import socketserver
import http.server
import urllib.request

HOST = '127.0.0.1'
PORT = 8080

class MyProxy(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        location=self.path[1:]
        
        # Rainviewer API documentation
        # https://www.rainviewer.com/api/weather-maps-api.html
        api = "https://api.rainviewer.com/public/weather-maps.json"
        resp = requests.get(url=api)
        host = resp.json()["host"]
        path = resp.json()["radar"]["past"][-1]["path"]
        size = 256
        color = 1 # Original color
        options = "1_1" # Smoothing and different snow colors
        
        url = f'{host}/{path}/{size}/{location}/{color}/{options}.png'
        
        self.send_response(200)
        self.end_headers()
        self.copyfile(urllib.request.urlopen(url), self.wfile)

requests_cache.install_cache(cache_name='rainviewer_cache', backend='sqlite', expire_after=300)
httpd = socketserver.ForkingTCPServer((HOST, PORT), MyProxy)
print("Now serving at: " + str(PORT))
httpd.serve_forever()
{% endhighlight %}

The settings within OsmAnd+ are fairly simple. In general, I've found that the short expiry time and a SQLiteDB file backend help to improve stability and prevent old tiles from displaying.

{:refdef: style="text-align: center; display: block; margin: 0 auto;"}
![Open Street Map online source settings](/static/OsmAndSettings.jpg){: width="200"}
{: refdef}
The OsmAnd+ online tile map settings I'm using.<br><br>
{: refdef}

Thank you for reading and definitely try out Open Street Map if you have the chance!