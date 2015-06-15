## Introduction ##

In the past, we had people complaining about the size of a library of **30KB** (which is completely ridiculous IMHO), and so to prevent such complains or to give you an overview of the minimun needed to send data to Google Analytcis servers, we will describe a step by step here.


## The Measurement Protocol ##

As you can see in the [Measurement Protocol Developer Guide](https://developers.google.com/analytics/devguides/collection/protocol/v1/devguide),<br>
to send data to Google Analytcis servers you need 3 things<br>
<br>
<ol><li>a client ID<br>
</li><li>well formated payload data<br>
</li><li>an HTTP POST or GET</li></ol>



<h2>The Client ID</h2>

As explained in <a href='https://developers.google.com/analytics/devguides/collection/protocol/v1/reference#required'>Required Values For All Hits</a>, you need 4 required values each time you send a request,<br>
all the rest is optional, but if one of those 4 is missing your request will fail.<br>
<br>
The <b>protocol version</b> is a piece of cake, just use <code>v=1</code>.<br>
<br>
The <b>Tracking ID</b> is quite easy too, you get it from the Google Analytics panel, something following this format <code>UA-XXXX-Y</code>.<br>
<br>
The <b>Hit Type</b> is one of the following strings: <code>pageview</code>, <code>screenview</code>, <code>event</code>, etc.<br>
(documented here <a href='https://developers.google.com/analytics/devguides/collection/protocol/v1/parameters#t'>Hit Type</a>).<br>
<br>
So, the real hard parameter to manage is the <b>Client ID</b><br>
(again documented here <a href='https://developers.google.com/analytics/devguides/collection/protocol/v1/parameters#cid'>Client ID</a>).<br>
<br>
The problem is not really to generate that client ID but to save it and reuse it.<br>
<br>
If for example you generate a new Client ID for each requests,<br>
the Google Analytics servers will see a different user for each request,<br>
something you really don't want.<br>
<br>
Another example would be tracking the same user from 2 different environments:<br>
on a web page the user would be tracked by <code>analytics.js</code><br>
and within that page a SWF file using <code>analytics.swc</code> would track also that user.<br>
<br>
In that case, if you use 2 different Client ID, you can not follow the flow of the user browsing.<br>
<br>
At the opposite, if you re-use the same Client ID, you can see where and when the user navigate from the HTML to the SWF (for example) etc.<br>
<br>
If in an HTML environment you would use Cookies,<br>
in a Flash environment we advise you to use the SharedObject class<br>
<pre><code>var so:SharedObject = SharedObject.getLocal( "_ga" );<br>
var cid:String;<br>
<br>
if( !_so.data.clientid )<br>
{<br>
    cid = generateUUID(); //not found so we generate it<br>
    so.data.clientid = cid;   //then we save it<br>
    so.flush( 1024 );<br>
}<br>
else<br>
{<br>
    cid = so.data.clientid; //found so we reuse it<br>
}<br>
</code></pre>

For the Client ID generation, the best method we know of is the implementation you can find here<br>
<code>com.google.analytics.utils.generateUUID()</code>.<br>
<br>
Why best ?<br>
<ul><li>it reuses <a href='http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/flash/crypto/package.html#generateRandomBytes()'>flash.crypto.generateRandomBytes</a> which<br>"The random sequence is generated using cryptographically strong functions provided by the operating system."<br>
</li><li>it generates a valid UUID v4<br>
</li><li>it follow to the letter the <a href='https://developers.google.com/analytics/devguides/collection/protocol/v1/parameters#cid'>Measurement Protocol Client ID</a> documentation<br>"The value of this field should be a random UUID (version 4) as described in <a href='http://www.ietf.org/rfc/rfc4122.txt'>http://www.ietf.org/rfc/rfc4122.txt</a>"</li></ul>

That said, you could use different way to generate this Client ID.<br>
<br>
Basically, it just need to be a random sequence to anonymously identify a particular user, device or browser instance<br>
we could then reuse what we were doing for <b>gaforflash</b>:<br>
<pre><code>function generate32bitRandom():int<br>
{<br>
    return Math.round( Math.random() * 0x7fffffff );<br>
}<br>
<br>
function generateHash( input:String ):int<br>
{<br>
    var hash:int      = 1;<br>
    var leftMost7:int = 0;<br>
    var pos:int;<br>
    var current:int;<br>
    <br>
    if(input != null &amp;&amp; input != "")<br>
    {<br>
        hash = 0;<br>
        <br>
        for( pos = input.length - 1 ; pos &gt;= 0 ; pos-- )<br>
        {<br>
            current   = input.charCodeAt(pos);<br>
            hash      = ((hash &lt;&lt; 6) &amp; 0xfffffff) + current + (current &lt;&lt; 14);<br>
            leftMost7 = hash &amp; 0xfe00000;<br>
            <br>
            if(leftMost7 != 0)<br>
            {<br>
                hash ^= leftMost7 &gt;&gt; 21;<br>
            }<br>
        }<br>
    }<br>
    <br>
    return hash;<br>
}<br>
<br>
function generateUserDataHash():Number<br>
{<br>
    var data:String = "";<br>
        data += Capabilities.cpuArchitecture;<br>
        data += Capabilities.language;<br>
        data += Capabilities.manufacturer;<br>
        data += Capabilities.os;<br>
        data += Capabilities.screenColor;<br>
        data += Capabilities.screenDPI;<br>
        data += Capabilities.screenResolutionX + "x" + Capabilities.screenResolutionY;<br>
<br>
    return generateHash( data );<br>
}<br>
<br>
function getClientID():Number<br>
{<br>
    var cid:Number = (generate32bitRandom() ^ generateUserDataHash()) * 0x7fffffff;<br>
    return cid;<br>
}<br>
</code></pre>


<h2>Well Formated Payload Data</h2>

It is all about following the rules of the <a href='https://developers.google.com/analytics/devguides/collection/protocol/v1/parameters'>Measurement Protocol Parameter Reference</a> but most importantly how to encode them correctly.<br>
<br>
It is described here <a href='https://developers.google.com/analytics/devguides/collection/protocol/v1/reference#encoding'>URL Encoding Values</a> with the following<br>
<hr />
All values sent to Google Analytics must be both UTF-8 and <a href='http://en.wikipedia.org/wiki/Percent-encoding'>URL Encoded</a>. To send the key <code>dp</code> with the value <code>/my page €</code>, you will first need to make sure this is UTF-8 encoded, then url encoded, resulting in the final string:<br>
<pre><code>dp=%2Fmy%20page%20%E2%82%AC<br>
</code></pre>
If any of the characters are encoded incorrectly, they will be replaced with the unicode replacement character <code>xFFFD</code>.<br>
<hr />

And then come ECMAScript history and ActionScript 3.0 ...<br>
<br>
In AS3 you have basically 3 different way to URL Encode stuff<br>
<ul><li>the <a href='http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/package.html#escape()'>escape()</a> function<br>Converts the parameter to a string and encodes it in a URL-encoded format, where most nonalphanumeric characters are replaced with <code>%</code> hexadecimal sequences.<br>
</li><li>the <a href='http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/package.html#encodeURI()'>encodeURI()</a> function<br>Encodes a string into a valid URI (Uniform Resource Identifier).<br>Converts a complete URI into a string in which all characters are encoded as UTF-8 escape sequences unless a character belongs to a small group of basic characters.<br>
</li><li>the <a href='http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/package.html#encodeURIComponent()'>encodeURIComponent()</a> function<br>Encodes a string into a valid URI component.<br>Converts a substring of a URI into a string in which all characters are encoded as UTF-8 escape sequences unless a character belongs to a very small group of basic characters.</li></ul>

and the nuance between them is subtle ...<br>
<br>
To keep a long story short, you really need to follow <a href='http://www.ietf.org/rfc/rfc2396.txt'>RFC 2396</a>, read a bit O'Reilly's "HTML: The Definitive Guide" (page 164), and then you understand you need to use <code>encodeURIComponent()</code>.<br>
<br>
For example, you would do that<br>
<pre><code>var payload:String = "";<br>
    payload += "dp=" + encodeURIComponent( "/my page €" );<br>
</code></pre>

A longer example would be<br>
<pre><code>var trackingID:String = "UA-123456-0";<br>
var clientID:String = generateUUID();<br>
<br>
var payload:Array = [];<br>
    payload.push( "v=1" );<br>
    payload.push( "tid=" + trackingID );<br>
    payload.push( "cid=" + clientID );<br>
    payload.push( "t=pageview" );<br>
    payload.push( "dh=mydomain.com" );<br>
    payload.push( "dp=" + encodeURIComponent( "/my page €" ) );<br>
    payload.push( "dt=" + encodeURIComponent( "My Page Title with €" ) );<br>
<br>
var data:String = payload.join( "&amp;" );<br>
</code></pre>


<h2>An HTTP POST or GET</h2>

In ActionScript 3.0 this will involve using a <code>URLRequest</code> with a <code>Loader</code> or a <code>URLLoader</code>.<br>
<br>
To keep things short and compatbile everywhere I would advise to use a <code>Loader</code> class (as with the <code>URLLoader</code> you can encounter security restriction) and send a <b>GET</b> request.<br>
<br>
<br>
<pre><code>var request:URLRequest = new URLRequest();<br>
    request.method = URLRequestMethod.GET;<br>
    request.url = "http://www.google-analytics.com/collect";<br>
    request.data = payload;<br>
<br>
var loader:Loader = new Loader();<br>
    loader.load( request );<br>
</code></pre>


<h2>The Final Result</h2>

A single class with a lot of traces that illustrate everything said above.<br>
<br>
There is a good reason if this class is not part of the source code, we don't plan to support it at all.<br>
<br>
Use it for testing, use it to understand how the Measurement protocol works, but then use it at your own risk.<br>
<br>
This class also illustrates why we do prefer to have a library instead, so we can have structure, tests, reusable code, etc.<br>
<br>
<pre><code>package test<br>
{<br>
    import flash.crypto.generateRandomBytes;<br>
    import flash.display.Loader;<br>
    import flash.events.ErrorEvent;<br>
    import flash.events.Event;<br>
    import flash.events.HTTPStatusEvent;<br>
    import flash.events.IOErrorEvent;<br>
    import flash.events.NetStatusEvent;<br>
    import flash.events.UncaughtErrorEvent;<br>
    import flash.net.SharedObject;<br>
    import flash.net.SharedObjectFlushStatus;<br>
    import flash.net.URLRequest;<br>
    import flash.net.URLRequestMethod;<br>
    import flash.utils.ByteArray;<br>
<br>
    public class SimplestTracker<br>
    {<br>
        private var _so:SharedObject;<br>
        private var _loader:Loader;<br>
        <br>
        public var trackingID:String;<br>
        public var clientID:String;<br>
        <br>
        public function SimplestTracker( trackingID:String )<br>
        {<br>
            trace( "SimplestTracker starts" );<br>
            this.trackingID = trackingID;<br>
            trace( "trackingID = " + trackingID );<br>
            <br>
            trace( "obtain the Client ID" );<br>
            this.clientID  = _getClientID();<br>
            trace( "clientID = " + clientID );<br>
        }<br>
        <br>
        private function onFlushStatus( event:NetStatusEvent ):void<br>
        {<br>
            _so.removeEventListener( NetStatusEvent.NET_STATUS, onFlushStatus);<br>
            trace( "User closed permission dialog..." );<br>
            <br>
            switch( event.info.code )<br>
            {<br>
                case "SharedObject.Flush.Success":<br>
                trace( "User granted permission, value saved" );<br>
                break;<br>
                <br>
                case "SharedObject.Flush.Failed":<br>
                trace( "User denied permission, value not saved" );<br>
                break;<br>
            }<br>
        }<br>
        <br>
        private function onLoaderUncaughtError( event:UncaughtErrorEvent ):void<br>
        {<br>
            trace( "onLoaderUncaughtError()" );<br>
            <br>
            if( event.error is Error )<br>
            {<br>
                var error:Error = event.error as Error;<br>
                trace( "Error: " + error );<br>
            }<br>
            else if( event.error is ErrorEvent )<br>
            {<br>
                var errorEvent:ErrorEvent = event.error as ErrorEvent;<br>
                trace( "ErrorEvent: " + errorEvent );<br>
            }<br>
            else<br>
            {<br>
                trace( "a non-Error, non-ErrorEvent type was thrown and uncaught" );<br>
            }<br>
            <br>
            _removeLoaderEventsHook();<br>
        }<br>
        <br>
        private function onLoaderHTTPStatus( event:HTTPStatusEvent ):void<br>
        {<br>
            trace( "onLoaderHTTPStatus()" );<br>
            trace( "status: " + event.status );<br>
            <br>
            if( event.status == 200 )<br>
            {<br>
                trace( "the request was accepted" );<br>
            }<br>
            else<br>
            {<br>
                trace( "the request was not accepted" );<br>
            }<br>
        }<br>
        <br>
        private function onLoaderIOError( event:IOErrorEvent ):void<br>
        {<br>
            trace( "onLoaderIOError()" );<br>
            _removeLoaderEventsHook();<br>
        }<br>
        <br>
        private function onLoaderComplete( event:Event ):void<br>
        {<br>
            trace( "onLoaderComplete()" );<br>
            <br>
            trace( "done" );<br>
            _removeLoaderEventsHook();<br>
        }<br>
        <br>
        private function _removeLoaderEventsHook():void<br>
        {<br>
            _loader.uncaughtErrorEvents.removeEventListener( UncaughtErrorEvent.UNCAUGHT_ERROR, onLoaderUncaughtError );<br>
            _loader.contentLoaderInfo.removeEventListener( HTTPStatusEvent.HTTP_STATUS, onLoaderHTTPStatus );<br>
            _loader.contentLoaderInfo.removeEventListener( Event.COMPLETE, onLoaderComplete );<br>
            _loader.contentLoaderInfo.removeEventListener( IOErrorEvent.IO_ERROR, onLoaderIOError );<br>
        }<br>
        <br>
        private function _generateUUID():String<br>
        {<br>
           var randomBytes:ByteArray = generateRandomBytes( 16 );<br>
            randomBytes[6] &amp;= 0x0f; /* clear version */<br>
            randomBytes[6] |= 0x40; /* set to version 4 */<br>
            randomBytes[8] &amp;= 0x3f; /* clear variant */<br>
            randomBytes[8] |= 0x80; /* set to IETF variant */<br>
            <br>
            var toHex:Function = function( n:uint ):String<br>
            {<br>
                var h:String = n.toString( 16 );<br>
                h = (h.length &gt; 1 ) ? h: "0"+h;<br>
                return h;<br>
            }<br>
            <br>
            var str:String = "";<br>
            var i:uint;<br>
            var l:uint = randomBytes.length;<br>
            randomBytes.position = 0;<br>
            var byte:uint;<br>
            <br>
            for( i=0; i&lt;l; i++ )<br>
            {<br>
                byte = randomBytes[ i ];<br>
                str += toHex( byte );<br>
            }<br>
            <br>
            var uuid:String = "";<br>
            uuid += str.substr( 0, 8 );<br>
            uuid += "-";<br>
            uuid += str.substr( 8, 4 );<br>
            uuid += "-";<br>
            uuid += str.substr( 12, 4 );<br>
            uuid += "-";<br>
            uuid += str.substr( 16, 4 );<br>
            uuid += "-";<br>
            uuid += str.substr( 20, 12 );<br>
            <br>
            return uuid;<br>
        }<br>
        <br>
        private function _getClientID():String<br>
        {<br>
            trace( "Load the SharedObject '_ga'" );<br>
            _so = SharedObject.getLocal( "_ga" );<br>
            var cid:String;<br>
            <br>
            if( !_so.data.clientid )<br>
            {<br>
                trace( "CID not found, generate Client ID" );<br>
                cid = _generateUUID();<br>
                <br>
                trace( "Save CID into SharedObject" );<br>
                _so.data.clientid = cid;<br>
                <br>
                var flushStatus:String = null;<br>
                try<br>
                {<br>
                    flushStatus = _so.flush( 1024 ); //1KB<br>
                }<br>
                catch( e:Error )<br>
                {<br>
                    trace( "Could not write SharedObject to disk: " + e.message );<br>
                }<br>
                <br>
                if( flushStatus != null )<br>
                {<br>
                    switch( flushStatus )<br>
                    {<br>
                        case SharedObjectFlushStatus.PENDING:<br>
                        trace( "Requesting permission to save object..." );<br>
                        _so.addEventListener( NetStatusEvent.NET_STATUS, onFlushStatus);<br>
                        break;<br>
                        <br>
                        case SharedObjectFlushStatus.FLUSHED:<br>
                        trace( "Value flushed to disk" );<br>
                        break;<br>
                    }<br>
                }<br>
                <br>
            }<br>
            else<br>
            {<br>
                trace( "CID found, restore from SharedObject" );<br>
                cid = _so.data.clientid;<br>
            }<br>
            <br>
            return cid;<br>
        }<br>
        <br>
        public function sendPageview( page:String, title:String = "" ):void<br>
        {<br>
            trace( "sendPageview()" );<br>
            <br>
            var payload:Array = [];<br>
                payload.push( "v=1" );<br>
                payload.push( "tid=" + trackingID );<br>
                payload.push( "cid=" + clientID );<br>
                payload.push( "t=pageview" );<br>
                /*payload.push( "dh=mydomain.com" ); */<br>
                payload.push( "dp=" + encodeURIComponent( page ) );<br>
                <br>
            if( title &amp;&amp; (title.length &gt; 0) )<br>
            {<br>
                payload.push( "dt=" + encodeURIComponent( title ) );    <br>
            }<br>
            <br>
            var url:String = "";<br>
                url += "http://www.google-analytics.com/collect";<br>
<br>
            var request:URLRequest = new URLRequest();<br>
                request.method = URLRequestMethod.GET;<br>
                request.url    = url;<br>
                request.data   = payload.join( "&amp;" );<br>
<br>
            trace( "request is: " + request.url + "?" + request.data );<br>
                <br>
            _loader = new Loader();<br>
            _loader.uncaughtErrorEvents.addEventListener( UncaughtErrorEvent.UNCAUGHT_ERROR, onLoaderUncaughtError );<br>
            _loader.contentLoaderInfo.addEventListener( HTTPStatusEvent.HTTP_STATUS, onLoaderHTTPStatus );<br>
            _loader.contentLoaderInfo.addEventListener( Event.COMPLETE, onLoaderComplete );<br>
            _loader.contentLoaderInfo.addEventListener( IOErrorEvent.IO_ERROR, onLoaderIOError );<br>
            <br>
            try<br>
            {<br>
                trace( "Loader send request" );<br>
                _loader.load( request );<br>
            }<br>
            catch( e:Error )<br>
            {<br>
                trace( "unable to load requested page: " + e.message );<br>
                _removeLoaderEventsHook();<br>
            }<br>
            <br>
        }<br>
    }<br>
}<br>
</code></pre>


usage:<br>
<pre><code>var tracker:SimplestTracker = new SimplestTracker( "UA-12345678-0" );<br>
    tracker.sendPageview( "/my page €", "My Page Title with €" );<br>
</code></pre>

This should work everywhere<br>
<ul><li>with a SWF running on localhost<br>
</li><li>with a SWF running on your own domain<br>
</li><li>with an AIR application (either Desktop or Mobile)</li></ul>

(it should, if it doesn't don't come asking for help "why it does not work ?", use the library instead)<br>
<br>
you should get an output similar to that:<br>
<pre><code>SimplestTracker starts<br>
trackingID = UA-12345678-0<br>
obtain the Client ID<br>
Load the SharedObject '_ga'<br>
CID found, restore from SharedObject<br>
clientID = 35009a79-1a05-49d7-b876-2b884d0f825b<br>
sendPageview()<br>
request is: http://www.google-analytics.com/collect?v=1&amp;tid=UA-12345678-0&amp;cid=35009a79-1a05-49d7-b876-2b884d0f825b&amp;t=pageview&amp;dp=%2Fmy%20page%20%E2%82%AC&amp;dt=My%20Page%20Title%20with%20%E2%82%AC<br>
Loader send request<br>
onLoaderHTTPStatus()<br>
status: 200<br>
the request was accepted<br>
onLoaderComplete()<br>
done<br>
</code></pre>


<h2>Conclusion</h2>

What we have described above is the strict minimum to "get it work", it is not something we plan to support or encourage.<br>
<br>
If you use the code above, instead of using the library, we will not help or support you in anyway, eg. <b>you are on your own</b>.<br>
<br>
It is our opinion that a library would be better suited to your needs (whatever the size this library is).