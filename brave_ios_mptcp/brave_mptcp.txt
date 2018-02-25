
# Extend Brave web browser on iOS to use Multipath-TCP for suggestions of search engines

Link to fork of the Brave repository:
https://github.com/slardinois/browser-ios

The initial goal was to use MPTCP for all features of Brave. However,
since the Webkit library is used for main features such as browsing, we cannot
extend those without changing either Webkit or the implementation of those
features.

Fortunately, Multipath-TCP can still  be used when Brave retrieves search engines
suggestions. It can be useful to have minimum latency for that, the user
expecting quickly a response.

## How to use Multipath-TCP

Multipath-TCP can be used only when `URLSessionConfiguration` is used. We can
specify that we want to use MPTCP by setting the attribute `multipathServiceType`.
We can set it to four different modes:

  - `none`: The default mode. MPTCP is not used for the session
  - `handout`: Provides a smooth handover between Wi-Fi and cellular network
  - `intercative`: The session will use the lowest-latency path
  - `aggregate`: Uses all the differents interfaces

In addition to that, you have to switch on the "Multipath" option of your
projects.

## Multipath-TCP and Brave

In the case of Brave iOS browser app, there are two places where
`URLSessionConfiguration` is used: in _Client/Frontend/Browser/SearchSuggestClient.swift_
and in _Utils/FaviconFetcher.swift_. We can also find a `URLSession` object in
_brave/src/webfilters/NetworkDataFileLoader.swift_ from which we can acces the
configuration object, but we did not manage to find what was its use in the code.
It is called at the start of the app and never again.

Here we will look at _SearchSuggestClient.swift_ but the following holds for
_FaviconFetcher.swift_. We will configure our connection to use the MPTCP
interactive mode in order to have low-latency suggestions.

Now we can configure the session to use MPTCP by adding the line

`configuration.multipathServiceType = .interactive`

right after

`let configuration = URLSessionConfiguration.ephemeral`

At that point, if you didn't set your project to target iOS 11, you should have
an error since MPTCP is available only for iOS 11 and above. You therefore have
two choices: either set your project to target iOS 11 at least, or you can change
the line we added by the following

    if #available(iOS 11.0, *) {
      configuration.multipathServiceType = .interactive
    }

And voila! Our connection to retrieve the suggestions of the search engines
now uses Multipath-TCP! You can check that by listening on the interface of the
phone (with tcpdump for example), you will see that the SYN flag contains the
option `mptcp capable` whenever you start typing on the URL bar.

However, even though the app now uses MPTCP, the servers of the search engines
probably don't use it. For the sake of the experiment we will now set up a
MPTCP capable proxy using _nginx_ that will forward every request to _Google_.

## Setting up a Multipath-TCP capable proxy

For this part you will need a MPCTP capable linux distribution (you can find it
[here](http://multipath-tcp.org)) and install _nginx_.

Once done, create a file called _proxy_ (the name actually does not matter) in
_/etc/nginx/_ and write the following in it:

    server {
      listen 8080;

      location / {
        resolver 8.8.8.8
        proxy_pass http://www.google.com;
      }
    }

then add a symbolic link as follow:

  `ln -s /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled/`

The server is now configured correctly, but you can change the port on which
it listens if you want to.

## Setting up Brave to use our proxy

Now that we have a MPTCP capable proxy, let's configure Brave to use it. To do
so, we will simply add our proxy in the search engine list of Brave.

So open the file _Client/Assets/Search/SearchPlugins/more/list.txt_ and ass
`mptcp` at the end. In the same folder, create a file called _mptcp.xml_
and copy/paste the content of _../en/google.xml_. Now open _mptcp.xml_ and change
the following:

  - Change the ShortName to `mptcp`
  - In the first `URL` marker, replace `www.google.com` in the template attribute
  by the IP address of your proxy server and the port it listens to. So for
  example it will be `100.101.102.103:8080"`
  - In the same template attribute, remove de `s` of `https`.
  (It now should look like this:
  `http://100.101.102.103:8080/complete/search/?client=firefox&amp;q={searchTerms}`)

Now in your app on your phone, you can change the settings to use the "mptcp"
search engine. And whenever you will type in the URL bar, the app will start a
MPTCP connection with our proxy server!
