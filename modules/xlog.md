# XLOG Module
The `xlog.module` (AKA *xlog*) is a bash module, designed to be *sourced* into other bash modules, and offers a set of functions that allow scripts to create/manage a structured log file.

## Log Model
The *xlog* functions assume the existence of a log file, where the messages are logged. The format of the logged messages is well defined, making the log file eligible for parsing.
The log file must be a regular linux file, and the account under which the _xlog_ is running must have read/write permissions on it, as write permissions on the folder where it stands.

### The Log Message
The message to be logged, is a single line message, trimmed and single-spaced.
If the original text does not conform to these characteristics, it will be formatted  accordingly.

For example, the text:
```
"    This     is a TeSt
message       "
```

will be displayed as:

`"This is a TeSt message"`

### The Message's Class
Each log message is bound to a class, which allow us to identify the nature of the message; default classes are:
 * _INF_, for informational messages
 * _WRN_, for warning messages
 * _ERR_, for error messages

Custom classes can be used, as far as they start with a letter followed by letters, digits, dash (-) or colon (:), and finish with a letter or digit, and in total, don't exceed 16 characters.
Example of custom classes, are:
```
 * INF:SETUP
 * CRITICAL
 * CRITICAL:COMMS
```

The class is case insensitive, meaning that _INF:COMMS_ is the same as _InF:CoMMs_.

### The Message Level
The message level decides which messages are written to the file and which not. The engine supports 10 logging levels, being level 0 the most basic level and level 9 the most complete.
When a log record is created, it is only written to the log file **iff** its level is less or equal than the current log level.
For example, if the engine is set to level 3, only messages of levels 0, 1, 2 and 3 are written to the log.

### Engine Start/Stop
The *xlog* engine starts as soon as it is initialized -- see <a href="#xlog-initialize">`xlog-initialize`</a>, and remains operational until the end of the script, **or** being terminated -- see <a href="#xlog-terminate">`xlog-terminate`</a>.

However, while initialized, the *xlog* engine, can be paused and/or resumed anytime it is required -- see <a href="#xlog-disable">`xlog-disable`</a> and <a href="#xlog-enable">`xlog-enable`</a>.
The engines preserves a disable count, which is incremented every time the _xlog_ is disabled. To re-enable it, the _xlog_ must be enabled the **same number of times** it was disabled.
This allows a safe usage on recursive functions.

For example:

~~~
function foo( ) {
    xlog-disable
    # do something here
    # ...
    
    if [[ ... ]]; then foo; fi
        
    # ...
    
    xlog-enable
}
~~~

## The Log File Format
The Log File is a text file, encoded in UTF-8 and using the Linux end of line specification (single line feed character).
Each log record is composed by one or more lines, being the first the record itself, and any optional lines that follows it, additional dumped data.

**Only text data can be dumped**

The following BNF describes a log file:
~~~
LOGFILE     ::= <BOM> <RECORD>*;
BOM         ::= \xEF \xBB \xBF;
RECORD      ::= <HEADER> <DATA>?;
HEADER      ::= <TSTAMP> ' [' <CLASS> '][' <LEVEL> '] ' <MESSAGE> <ANCHOR><EOL>;
DATA        ::= '> ' <TEXT> <EOL>;
TSTAMP      ::= <<UTC date/time in the format YYYY-MM-DDThh:mm:ss.nnnZ>>;
CLASS       ::= [a-zA-Z]([a-zA-Z0-9:\-]*[a-zA-Z0-9])?;
LEVEL       ::= [0-9];
MESSAGE     ::= [\x21-\x7e]((\x20|[\x21-\x7e]+)*[\x21-\x7e])?;
ANCHOR      ::= ' (function "' <TEXT> '" on script ' <TEXT> ', line ' <NUMBER>;
TEXT        ::= [\x20-\x7e]+;
NUMBER      ::= [0-9]+;
EOL         ::= \x10;
~~~

Example of a log file extract:

```
2021/02/18 11:28:02.086Z [INF             ][0] xlog initialized (function "main" on script /usr/local/sbin/server.sh, line 23)
2021/02/18 11:48:24.373Z [INF             ][0] xlog level changed to 1 (function "main" on script /usr/local/sbin/server.sh, line 24)
2021/02/18 12:47:14.407Z [INF             ][0] Connecting to server xxx.com (function "" on script /usr/local/sbin/server.sh, line 312)
2021/02/18 12:47:37.299Z [INF:COMMS       ][0] Server response: OK (function "connect" on script /usr/local/sbin/server.sh, line 321)
2021/02/18 12:47:54.886Z [ERR:COMMS       ][0] Timeout (function "connect" on script /usr/local/sbin/server.sh, line 345)
2021/02/18 12:50:39.777Z [ERR:COMMS       ][1] Debug Info (function "connect" on script /usr/local/sbin/server.sh, line 346)
> Connect Status: OK
> Send Command  : OK
> Recv Response : TIMEOUT [5 secs]
2021/02/18 12:51:31.940Z [WRN:COMMS       ][0] Retry (function "connect" on script /usr/local/sbin/server.sh, line 350)
```

On the example above, the 6th line was generated at 12:50:39.777 UTC time, from 2021/02/18; it has the class ERR:COMMS, and a log level of 1.
The log message is `Debug Info` and was triggered by the script `server.sh`, located at `/usr/locals/bin`, by the function `connect`, on line 346.
The log dumped additional data (the 3 following lines that start with `>`).

## API

The _xlog_ module exposes the following functions:
 * <a href="#xlog-initialize">`xlog-initialize`</a>
 * <a href="#xlog-terminate">`xlog-terminate`</a>
 * <a href="#xlog-get-state">`xlog-get-state`</a>
 * <a href="#xlog-get-logfile">`xlog-get-logfile`</a>
 * <a href="#xlog-enable">`xlog-enable`</a>
 * <a href="#xlog-disable">`xlog-disable`</a>
 * <a href="#xlog-get-level">`xlog-get-level`</a>
 * <a href="#xlog-set-level">`xlog-set-level`</a>
 * <a href="#xlog-write">`xlog-write`</a>

<div id="xlog-initialize">

### xlog-initialize

**Description**
This function initializes the *xlog* module.

**Syntax** 
```
    xlog-initialize <log file> <flags>*
```

**Parameters**

<table>
 <tr>
  <th width="5em">Name</th>
  <th width="25em">Description</th>
 </tr>
 <tr>
  <td valign="top"><tt>&lt;log file&gt;</tt></td>
  <td>Path to the log path</td>
 </tr>
 <tr>
  <td valign="top"><tt>&lt;flags&gt;</tt></td>
  <td>Optional flags that specify how the _xlog-initialize_ should behave; Possible values are:<br/><br/>
      <table>
	    <tr><th width="10em">Value</th><th>Description</th></tr>
		<tr><td><tt>force</tt></td><td>Forces the re-initialization of the <i>xlog</i> engine when this was already initialized and/or enforces the overwrite of the log file when specifying this is found to be faulty</td></tr>
		<tr><td><tt>silent</tt></td><td>Does not add an INF message specifying that an initialization took place, when the log file initialization completes</td></tr>
	  </table>
  </td>
 </tr>
</table>

**Remarks**
The `xlog-initialize` initialized the xlog engine. If the engine was already initialized, and the flag _force_ is not specified, the command fails. Otherwise, it calls `xlog-terminate` prior to run the initialization code.
The initialization code starts by checking if the specified log file exists. If the file exists, checks that:
 * Is a regular file
 * It has read and write access to it
 * If not empty, it must start with a UTF-8 BOM sequence
 * Every line must be a log record entry **or** a data dump record
 * Check if the file ends with a _line feed_ character (ASCII `\x0A`). If not, adds the following _re-synchronization_ message ` *** re-synchronizing log file` (remark: notice the **blank** before the `***`)

If the file does not exist, it creates one with the UTF-8 BOM

If the file already exists but does not have a UTF-8 BOM **or** it's structure is not correct, it checks for the `force` flag. If not present, it fails the initialization; otherwise, backup the old file and create a new log file.
The log file backup has the same name as the original, appended with the extension `.bak_YYYYMMDDhhmmss`, where `YYYYMMDDhhmmss` is the current UTC timestamp.

At the end of the initialization, it checks for the *silent* flag; if not present, it adds a INF level 0 message, specifying `xlog initialized`. If `silent` is specified, the message is not added.

**Return**
N/A

**Return Codes**
<table>
 <tr>
  <th>Return Code</th>
  <th>Description</th>
 </tr>
 <tr><td align="right">0</td><td>Initialization successful</td>
 <tr><td align="right">1</td><td>No log file specified</td></tr>
 <tr><td align="right">2</td><td>Invalid flag</td></tr>
 <tr><td align="right">3</td><td><i>xlog</i> already initialized and <i>forced</i> not specified</td></tr>
 <tr><td align="right">4</td><td>The specified log file is not a regular file or is not readable or writable</td>
 <tr><td align="right">5</td><td>Can't create/update the specified log file</td>
 <tr><td align="right">6</td><td>No UTF-8 BOM found</td>
 <tr><td align="right">7</td><td>Log file integrity error</td>
</table>
</div>
<div id="xlog-terminate">

**Example**

- Initializes the *xlog* module to use file `/tmp/setup.log`
  ```bash
  xlog-initialize /tmp/setup
  ```
  
- Initializes the *xlog* module to use file `/tmp/setup.log` without logging the initialization message 
  ```bash
  xlog-initialize /tmp/setup silent
  ``` 
</div>

<div id="#xlog-terminate">

### xlog-terminate

**Description**
This function terminates the *xlog* module.

**Syntax** 
```
    xlog-terminate silent?
```

**Parameters**

<table>
 <tr>
  <td width="5em"><b>Name</b></td>
  <td width="25em"><b>Description</b></td>
 </tr>
 <tr>
  <td valign="top"><tt>silent</tt></td>
  <td>Does not add an INF message when the log file terminates completes, specifying that an cleanup took place.</td>
 </tr>
</table>

**Remarks**
The `xlog-terminate` cleanups the _xlog_ engine.
At the end of the cleanup, it checks for the _silent_ flag; if not present, it adds a INF level 0 message, specifying ```xlog terminated```. If _silent_ is specified, the message is not added.

**Return**
N/A

**Return Codes**
<table>
 <tr>
  <th><b>Return Code</b></th>
  <th><b>Description</b></th>
 </tr>
 <tr><td align="right">0</td><td>cleanup successful</td>
 <tr><td align="right">1</td><td><i>xlog</i> is not initialized</td></tr>
 <tr><td align="right">2</td><td>invalid flag</td></tr>
</table>
</div> 

**Example**

- cleanup the *xlog* engine
   ```bash
  xlog-terminate
  ```

- silently, cleanup the *xlog* engine
  ```bash
  xlog-terminate silent
  ```

<div id="xlog-get-state">

### xlog-get-state

**Description**
This function returns the *xlog* state.

**Syntax** 
```
    xlog-get-state
```

**Parameters**
N/A

**Remarks**
N/A

**Return**

The following values are sent to the `stdout`:
<table>
 <tr><th>Value</th><th>Condition</th></tr>
 <tr><td><tt>uninitialized</tt></td><td>The <i>xlog</i> is not initialized</td></tr>
 <tr><td><tt>disabled</tt></td><td>The <i>xlog</i> is disabled (see <a href="#xlog-disable"><tt>xlog-disable</tt></a>)</td></tr>
 <tr><td><tt>enabled</tt></td><td>The <i>xlog</i> is enabled (see <a href="#xlog-enable"><tt>xlog-enable</tt></a>)</td></tr>
</table>

**Return Codes**
0

**Example**

- Check if the *xlog* engine is enabled
  ```bash
  if [[ `xlog-get-state` == enabled ]]; then do-something; fi
  ```

</div>

<div id="xlog-get-logfile">

### xlog-get-logfile

**Description**
This function returns the current log file.

**Syntax** 
```
    xlog-get-logfile
```

**Parameters**
N/A

**Remarks**
N/A

**Return**
On success, writes the full path of the log file on the `stdout`.

**Return Codes**
<table>
 <tr>
  <th><b>Return Code</b></th>
  <th><b>Description</b></th>
 </tr>
 <tr><td align="right">0</td><td>Success</td>
 <tr><td align="right">1</td><td><i>xlog</i> not initialized</td></tr>
</table>

**Example**

- shows the log file
  ```bash
  echo `xlog-get-logfile`
  ```

</div>

<div id="xlog-enable">

### xlog-enable

**Description**
This function enables the *xlog* engine.

**Syntax** 
```
    xlog-enable silent?
```

**Parameters**
<table>
 <tr>
  <th width="5em">Name</th>
  <th width="25em">Description</th>
 </tr>
 <tr>
  <td valign="top"><tt>silent</tt></td>
  <td>Does not add a INF message in case the module becomes enabled nor a WRN message when on the attempt to enable an already enabled engine.</td>
 </tr>
</table>

**Remarks**
The `xlog-enable`, decrements the disable count (see <a href="#xlog-disable">`xlog-disable`</a>); When this count reaches 0, the <i>xlog</i> writes a INF level 0 on the log, informing that the engine is now enabled.

If `xlog-enable` is called when the module is already enabled, it logs a WRN level 0 message, informing that there was an attempt to enable an enabled module.

**Return**
N/A

**Return Codes**
<table>
 <tr>
  <th><b>Return Code</b></th>
  <th><b>Description</b></th>
 </tr>
 <tr><td align="right">0</td><td>Enable command successful</td>
 <tr><td align="right">1</td><td><i>xlog</i> engine not initialized</td></tr>
 <tr><td align="right">2</td><td>invalid flag</td></tr>
 <tr><td align="right">3</td><td><i>xlog</i> the engine is already enabled.</td></tr>
</table>

**Example**

- Enable the log engine
  ```bash
  xlog-enable
  ```
  
</div>
<div id="xlog-disable">

### xlog-disable

**Description**
This function disables the *xlog* engine.

**Syntax** 
```
    xlog-disable silent?
```

**Parameters**
<table>
 <tr>
  <th width="5em">Name</th>
  <th width="25em">Description</th>
 </tr>
 <tr>
  <td valign="top"><tt>silent</tt></td>
  <td>Does not add a INF message when the module becomes disabled.</td>
 </tr>
</table>

**Remarks**
The `xlog-disable`, increments the disable count (see <a href="#xlog-enable">`xlog-enable`</a>); When this count changes from 0 to 1, the <i>xlog</i> writes a WRN level 0 on the log, informing that the engine is now disabled.

Calling `xlog-disable` multiple times continuously increments the disable count. To re-enable a disabled module, <a href="#xlog-enable">`xlog-enable`</a> needs to be called, as many times as `xlog-disable` was called.
This allows the use of disable/enable in recursive functions. For example:
```bash
function foo( ) {
    # disable xlog while executing foo
    xlog-disable
    # ...
    if [[ condition ]]; then foo; fi
    # ...
    xlog-enable
}
```

**Return**
N/A

**Return Codes**
<table>
 <tr>
  <th>Return Code</th>
  <th>Description</th>
 </tr>
 <tr><td align="right">0</td><td>Enable command successful</td>
 <tr><td align="right">1</td><td><i>xlog</i> engine not initialized</td></tr>
</table>
</div>
<div id="xlog-get-level">

### xlog-get-level

**Description**
This function returns the current *xlog* log level.

**Syntax** 
```
    xlog-get-level
```

**Parameters**
N/A

**Remarks**
Only messages which log level is less or equal to the *xlog* log level are logged.

**Return**
Writes the current log level to the `stdout`.

**Return Codes**
<table>
 <tr>
  <th>Return Code</th>
  <th>Description</th>
 </tr>
 <tr><td align="right">0</td><td>Operation successful</td>
 <tr><td align="right">1</td><td><i>xlog</i> not initialized</td></tr>
</table>

**Example**

- Retrieve the current log level
  ```bash
  local logLevel=`xlog-get-loglevel`
  ```

</div>
<div id="xlog-set-level">

### xlog-set-level

**Description**
This function sets the current initializes the *xlog* module.

**Syntax** 
```
    xlog-set-level <level>
```

**Parameters**

<table>
 <tr>
  <th width="5em">Name</th>
  <th width="25em">Description</th>
 </tr>
 <tr>
  <td valign="top"><tt>&lt;level&gt;</tt></td>
  <td>New log level; must be between 0 and 9 (incl.)</td>
 </tr>
 <td valign="top"><tt>silent</tt></td>
  <td>if present, no notification is logged</td>
 </tr>
</table>

**Remarks**
When changing the log level, if:
 * `silent` was not specified, and
 * the new level is different from the current one

Then, an INF level 0 message is written on the log, informing the new log level.

**Return**
N/A

**Return Codes**
<table>
 <tr>
  <th><b>Return Code</b></th>
  <th><b>Description</b></th>
 </tr>
 <tr><td align="right">0</td><td>New log level applied</td>
 <tr><td align="right">1</td><td>_xlog_ not initialized</td></tr>
 <tr><td align="right">2</td><td>invalid level</td></tr>
</table>

**Example**

- Increase the current log level by 1
  ```bash
  xlog-set-level $((`xlog-get-level` + 1))
  ```
  
</div>
<div id="xlog-write">

### xlog-write

**Description**
This function writes a new log message

**Syntax** 
```
    xlog-write <message> (<type> (<level> <dump>?)?)?
```

**Parameters**

<table>
 <tr>
  <th width="5em">Name</th>
  <th width="25em">Description</th>
 </tr>
 <tr>
  <td valign="top"><tt>&lt;message&gt;</tt></td>
  <td>Message to be logged</td>
 </tr>
 <tr>
  <td valign="top"><tt>&lt;type&gt;</tt></td>
  <td>Message type; this is a 16 character string. Default used values are:
    <table>
      <tr><th>Value</th><th>Description</th></tr>
      <tr><td><tt>INF</tt></td><td>Information message</td></tr>
      <tr><td><tt>WRN</tt></td><td>Warning message</td></tr>
      <tr><td><tt>ERR</tt></td><td>Error message</td></tr>
     </table>
     <br/>If no class is specified, <tt>INF</tt> is assumed.
  </td>
 </tr>
 <tr><td><tt>level</tt></td><td>Message level; a value between 0 and 9 (incl.).<br/>If no <tt>level</tt> is specified, 0 is assumed.</td></tr>
 <tr><td><tt>dump</tt></td><td>A stream to data to be dumped or - for the <tt>stdout</tt>. The data should be text.<br/>If no stream is specified, nothing will be dumped.</td></tr>
</table>

**Remarks**

***The Message***
The message should be a single line, trimmed and single spaced text message. This means that:
 * any space in the beginning and/or end of the message will be stripped
 * multiple blanks (spaces **and** tabs) in the middle of the message, will be converted to a single space
 * any control character will be stripped

For example, the message:
```
"     Hello     wo
rld!          Life is beautiful                 "
```
will be converted to:
```
"Hello wo rdl! Life is beautiful"
```
Note the space in between the word <i>world</i>. This happens because the word was split.

***The Message Class***
The message class is used to classify the type of message that is being logged. Despite the fact that the *xlog* uses `INF`, `WRN` and `ERR` for informational, warning and error messages, any class can be used.

The only rules imposed to classes are:
 * They must start with a letter
 * They must end with a letter or a digit
 * In between, they can have letters, digits, colon (:) and dashes (-)
 * In total, classes cannot have more than 16 characters
 * Classes are case insensitive

Example of classes:
```
  - INF
  - INF:COMMS
  - CRITICAL
  - WRN:SETUP-INIT
```

***The Message Level***
The message level is used to support multiple types of log detail. A message is written to the log is its level is less or equal to the _xlog_ log level (see <a href="#xlog-set-level">xlog-set-level</a> and <a href="#xlog-get-level">xlog-get-level</a>).

***The Data to Dump***
The data to be dumped will appear under the log record, preceded by &gt;
Binary data will be stripped from the dump.

If a message is not written because the _xlog_ is disabled or the message level is higher than the log level but a stream is provided, the stream will be equally consumed, but discarded.
This ensures the same behavior in what respected the consumption of data to be dumped.

**Return**
N/A

**Return Codes**
<table>
 <tr>
  <th><b>Return Code</b></th>
  <th><b>Description</b></th>
 </tr>
 <tr><td align="right">0</td><td>Operation successful. <br/>If the <i>xlog</i> is disabled, or the message level is greater than the current log level, the log entry <b>will not</b> be written, but this operation is still considered successful</td>
 <tr><td align="right">1</td><td><i>xlog</i> not initialized</td></tr>
 <tr><td align="right">2</td><td>no message provided</td></tr>
 <tr><td align="right">3</td><td>bad class</tr>
 <tr><td align="right">4</td><td>bad level</tr>
</table>

**Example**

- Write a INF level 0 "Hello, world" message 
  ```bash
  xlog-write "Hello, world"
  ```
- Write a WRN level 2 "Hello, world" message
  ```bash
  xlog-write "Hello, world" WRN 2
  ```

- Write a message, using class `INF:SETUP` and level 1, dumping the outcome of the command `connect`
  ```bash
  connect | xlog-write "Connecting to server" 'INF:SETUP' 1 -
  ```

- Write a log message, dumping the outcome of  `ls /var -Rl`, as INF level 0
  ```bash
  xlog-write "Dump of /var and sub-directories" "" "" <(ls /var -Rl)
  ```

</div>