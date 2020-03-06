## Introduction

In order to serve User directory (User Authentication/Authorization) from a PostgreSQL database we will use following approach

1.  mod\_lua bindings for serving User directory, a LUA script will take care of database integration and other details
2.  Users will be stored as a JSON object in a JSON type column, PostgreSQL supports &quot;JSON&quot; columns
3.  3rd party postgres-json-schema extension to validate JSON schema elements

Basic syntax validation of JSON document is done by PostgreSQL server itself, however to avoid pitfalls and ensure data integry we need to validate FreeSWITCH related elements in JSON doc before inserting/updating, unfortunately PostgreSQL doesn&#39;t support schema validation i.e. validating a JSON document against a JSON schema. In order to accomplish this we will use a third party PostgreSQL extension named &quot;postgres-json-schema&quot;, once installed it provides us a function `validate_json_schema(schema, data)` that returns true if data validates against the schema and false if not

## Prerequisites

Make sure following requirements are already met

*   FreeSWITCH 1.8 or above is already installed, (mod_lua is loaded)
*   Core (switch.conf.xml) and internal sip profile (sip_profiles/internal.xml) db is already set to ODBC DSN
*   PostgreSQL server  is installed and running

## Installation Steps

#### Install PostgreSQL development package

```bash
sudo apt install -y postgresql-server-dev-12
```

#### Clone &quot;postgres-json-schema&quot; repository

```bash
cd /usr/local/src/
git clone https://github.com/gavinwahl/postgres-json-schema.git
```

#### Install &quot;postgres-json-schema&quot; extension

Change to `postgres-json-schema` directory and run `make install`, the output should look like this

```bash
[root@master ~]# cd postgres-json-schema/
[root@master postgres-json-schema]# make install
/usr/bin/mkdir -p '/usr/pgsql-10/share/extension'
/usr/bin/mkdir -p '/usr/pgsql-10/share/extension'
/usr/bin/install -c -m 644 .//postgres-json-schema.control '/usr/pgsql-10/share/extension/'
/usr/bin/install -c -m 644 .//postgres-json-schema--0.1.0.sql  '/usr/pgsql-10/share/extension/'
```

#### Load &quot;postgres-json-schema&quot; extension

Connect to the database to create the “`postgres-json-schema`” extension. Extension creates “`validate_json_schema`” Pl/PgSQL function, which can be called against JSON data column in CHECK constraint. You can also use a GUI tool like PgAdmin4&#39;s Query Tool

```bash
[root@master]# su postgres -
[postgres@master]# psql 
psql (12)
Type "help" for help.

postgres=# \dx
                 List of installed extensions
  Name   | Version |   Schema   |         Description
---------+---------+------------+------------------------------
 plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
(1 row)

postgres=# CREATE EXTENSION "postgres-json-schema";
CREATE EXTENSION
```

#### Create extensions table for User Directory

Create a table in PostgreSQL to store FreeSWITCH Directory users, Our LUA script will query this table to fetch user information

```sql
CREATE TABLE extensions
(
    extension character varying(10) COLLATE pg_catalog."default" NOT NULL,
    json_data jsonb,
    CONSTRAINT extensions_pkey PRIMARY KEY (extension),
)
```

Create a CHECK constraint on above created table, this CHECK constraint will validate json\_data field on INSERT/UPDATE using provided schema, You may add more properties to schema anytime

```sql

ALTER TABLE extensions ADD CONSTRAINT data_is_valid CHECK (validate_json_schema('{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "title": "FreeSWITCH JSON Schema for User Directory",
    "description": "In case you are going to put your SIP User inside a postgres jsonb field, you will need this schema", 
    "type": "object",
    "properties": {
      "id": { "type": "integer" },
      "params": {
        "type": "object",
        "properties": {
          "password": { "type": "string" },
          "a1-hash": { "type": "string" },
          "dial-string": { "type": "string" },
          "vm-password": { "type": "string" },
          "vm-enabled": { "type": "string" },
          "vm-mailto": { "type": "string" },
          "vm-email-all-messages": { "type": "string" },
          "vm-notify-all-messages": { "type": "string" },
          "vm-attach-file": { "type": "string" },
          "jsonrpc-allowed-methods": { "type": "string" },
          "jsonrpc-allowed-event-channels": { "type": "string" }
        },
        "oneOf": [text
          { "required": [ "password" ] },
          { "required": [ "a1-hash" ] }
        ],
        "additionalProperties": false
      },
      "variables": {
        "type": "object",
        "properties": {
          "user_context": { "type": "string" },
          "callgroup": { "type": "string" },
          "sched_hangup": { "type": "string" },
          "toll_allow": { "type": "string" },
          "accountcode": { "type": "string" },
          "nibble_account": { "type": "string" },
          "origination_caller_id_name": { "type": "string" },
          "origination_caller_id_number": { "type": "string" },
          "effective_caller_id_name": { "type": "string" },
          "effective_caller_id_number": { "type": "string" },
          "outbound_caller_id_name": { "type": "string" },
          "outbound_caller_id_number": { "type": "string" }
        },
        "required": [ "user_context", "callgroup", "accountcode" ],
        "additionalProperties": true
      }
    },
    "required": [ "id", "params", "variables" ]
}', json_data));
```

Now, postgres will reject any INSERTs or UPDATES to the table that set data to anything not valid against its schema, Insert a sample user record to test if everything is working

```sql
INSERT INTO public.extensions(
	extension, json_data)
	VALUES (7001, '{
    "id": 7001,
    "params": {
        "password": "7001",
        "vm-mailto": "g.mustafa@emergen.info",
        "vm-password": "123"
    },
    "variables": {
        "callgroup": "vas",
        "accountcode": "33579",
        "user_context": "default"
    }
}');
```

#### Configure `mod_lua` bindings

The file `autoload_configs/lua.conf.xml` is used in the default FreeSWITCH setup. Here is a minimum configuration file, it will fetch User directory from Lua script.

```xml

<configuration name="lua.conf" description="LUA Configuration">
  <settings>
    <param name="xml-handler-script" value="user_directory.lua"/>
    <param name="xml-handler-bindings" value="directory"/>
  </settings>
</configuration>
```

> When you edit lua.conf.xml, giving the reloadxml command from the FreeSWITCH console is not sufficient to recognize the &quot;xml-handler-script&quot; parameters, you have to restart FreeSWITCH.

#### Create LUA script

Our LUA Script is the heart of this integration task. This script will be executed whenever FreeSWITCH requires a user information e.g for registering and calling. Create/Open a new file `user_directory.lua` inside `/usr/local/freeswitch/scripts` in your favorite editor and insert following content

```lua
-- Load required libraries 
package.path = '/usr/local/freeswitch/scripts/json.lua'
json = require("json")

package.path = '/usr/local/freeswitch/scripts/common.lua'
common = require("common")

-- temp logging
-- freeswitch.consoleLog("debug", "[xml_handler] KEY_NAME: " .. XML_REQUEST['key_name'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] KEY_VALUE: " .. XML_REQUEST['key_value'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] SECTION: " .. XML_REQUEST['section'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] TAG_NAME: " .. XML_REQUEST['tag_name'] .. "");
-- freeswitch.consoleLog("debug", "[xml_handler] Params: " .. params:serialize() .. "");

-- Get User from Request
user = params:getHeader("user")
domain_name = params:getHeader("domain")
event_calling_func = params:getHeader("Event-Calling-Function")

-- Get database connection handl3e
db = freeswitch.Dbh("odbc://freeswitch:freeswitch:freeswitch!")
-- db = freeswitch.Dbh("pgsql://hostaddr=127.0.0.1 dbname=freeswitch user=freeswitch password='freeswitch!' options='-c client_min_messages=NOTICE' application_name='freeswitch'")
-- check if we are connected to the database
assert(db:connected())

if common.is_user_directory_function(event_calling_func) then
	query = string.format("select extension, json_data::text as json_data from extensions where extension = '%s'", user)
	freeswitch.consoleLog("debug", "User query: " .. query);

	function fetch_user_object(row)
		json_data = row.json_data
		extension = row.extension
	end

	db:query(query, fetch_user_object)

	if (json_data == nil) then
		XML_STRING = common.XML_STRING_NOT_FOUND
	end

	if (json_data) then 
	    freeswitch.consoleLog("debug", "JSON:Data -> " .. json_data)
	    lua_value = json.parse(json_data) -- Get lua object from json object

	    lua_value['params']['dial-string']  = "{^^:sip_invite_domain=${dialed_domain}:presence_id=${dialed_user}@${dialed_domain}}${sofia_contact(*/${dialed_user}@${dialed_domain})},${verto_contact(${dialed_user}@${dialed_domain})}";
	    call_group = lua_value['variables']['call_group']
	    if call_group == nil then
		    call_group = user
	    end
            local xml = {}
            table.insert(xml, [[<document type="freeswitch/xml">]]);
            table.insert(xml, [[  <section name="directory">]]);
            table.insert(xml, [[    <domain name="]] .. domain_name .. [[">]]);
            table.insert(xml, [[      <groups>]]);
            table.insert(xml, [[        <group name="]] .. call_group .. [[">]]);
            table.insert(xml, [[           <users>]]);
            table.insert(xml, [[             <user id="]] .. user .. [[">]]);
            for block, bvalue in pairs(lua_value) do 
                if type(bvalue) == 'table' then
                        table.insert(xml, [[               <]].. block  ..[[>]]);
                for tag, tvalue in pairs(bvalue) do
                            table.insert(xml, [[                 <]].. string.sub(block, 1, -2)  ..[[ name="]] .. tag .. [[" value="]] .. tvalue .. [["/>]]);
                end
                        table.insert(xml, [[               </]].. block  ..[[>]]);
                end
            end
            table.insert(xml, [[             </user>]]);
            table.insert(xml, [[           </users>]]);
            table.insert(xml, [[        </group>]]);
            table.insert(xml, [[      </groups>]]);
            table.insert(xml, [[    </domain>]]);
            table.insert(xml, [[  </section>]]);
            table.insert(xml, [[</document>]]);

            XML_STRING = table.concat(xml, "\n");
	end
else
		XML_STRING = common.XML_STRING_NOT_FOUND
end -- user directory event calling funcs

freeswitch.consoleLog("debug", XML_STRING)
```

####   

#### Create `common.lua`

```lua
local common = {};
common.title = "Welcome to common functions"

common.user_directory_funcs = {'user_outgoing_channel', 'sofia_reg_parse_auth', 'voicemail_leave_main', 'user_data_function'}
common.sofia_funcs = {'config_sofia'}

function common.is_user_directory_function (val)
    for index, value in ipairs(common.user_directory_funcs) do
        if value == val then
            return true
        end
    end
    return false
end

common.XML_STRING_NOT_FOUND = [[
    <?xml version="1.0" encoding="UTF-8" standalone="no"?>
    <document type="freeswitch/xml">
                    <section name="result">
                            <result status="not found" />
                    </section>
            </document>
]];

return common
```

#### Create `json.lua`

```lua
--[[ json.lua

local json = {}


-- Internal functions.

local function kind_of(obj)
  if type(obj) ~= 'table' then return type(obj) end
  local i = 1
  for _ in pairs(obj) do
    if obj[i] ~= nil then i = i + 1 else return 'table' end
  end
  if i == 1 then return 'table' else return 'array' end
end

local function escape_str(s)
  local in_char  = {'\\', '"', '/', '\b', '\f', '\n', '\r', '\t'}
  local out_char = {'\\', '"', '/',  'b',  'f',  'n',  'r',  't'}
  for i, c in ipairs(in_char) do
    s = s:gsub(c, '\\' .. out_char[i])
  end
  return s
end

-- Returns pos, did_find; there are two cases:
-- 1. Delimiter found: pos = pos after leading space + delim; did_find = true.
-- 2. Delimiter not found: pos = pos after leading space;     did_find = false.
-- This throws an error if err_if_missing is true and the delim is not found.
local function skip_delim(str, pos, delim, err_if_missing)
  pos = pos + #str:match('^%s*', pos)
  if str:sub(pos, pos) ~= delim then
    if err_if_missing then
      error('Expected ' .. delim .. ' near position ' .. pos)
    end
    return pos, false
  end
  return pos + 1, true
end

-- Expects the given pos to be the first character after the opening quote.
-- Returns val, pos; the returned pos is after the closing quote character.
local function parse_str_val(str, pos, val)
  val = val or ''
  local early_end_error = 'End of input found while parsing string.'
  if pos > #str then error(early_end_error) end
  local c = str:sub(pos, pos)
  if c == '"'  then return val, pos + 1 end
  if c ~= '\\' then return parse_str_val(str, pos + 1, val .. c) end
  -- We must have a \ character.
  local esc_map = {b = '\b', f = '\f', n = '\n', r = '\r', t = '\t'}
  local nextc = str:sub(pos + 1, pos + 1)
  if not nextc then error(early_end_error) end
  return parse_str_val(str, pos + 2, val .. (esc_map[nextc] or nextc))
end

-- Returns val, pos; the returned pos is after the number's final character.
local function parse_num_val(str, pos)
  local num_str = str:match('^-?%d+%.?%d*[eE]?[+-]?%d*', pos)
  local val = tonumber(num_str)
  if not val then error('Error parsing number at position ' .. pos .. '.') end
  return val, pos + #num_str
end


-- Public values and functions.

function json.stringify(obj, as_key)
  local s = {}  -- We'll build the string as an array of strings to be concatenated.
  local kind = kind_of(obj)  -- This is 'array' if it's an array or type(obj) otherwise.
  if kind == 'array' then
    if as_key then error('Can\'t encode array as key.') end
    s[#s + 1] = '['
    for i, val in ipairs(obj) do
      if i > 1 then s[#s + 1] = ', ' end
      s[#s + 1] = json.stringify(val)
    end
    s[#s + 1] = ']'
  elseif kind == 'table' then
    if as_key then error('Can\'t encode table as key.') end
    s[#s + 1] = '{'
    for k, v in pairs(obj) do
      if #s > 1 then s[#s + 1] = ', ' end
      s[#s + 1] = json.stringify(k, true)
      s[#s + 1] = ':'
      s[#s + 1] = json.stringify(v)
    end
    s[#s + 1] = '}'
  elseif kind == 'string' then
    return '"' .. escape_str(obj) .. '"'
  elseif kind == 'number' then
    if as_key then return '"' .. tostring(obj) .. '"' end
    return tostring(obj)
  elseif kind == 'boolean' then
    return tostring(obj)
  elseif kind == 'nil' then
    return 'null'
  else
    error('Unjsonifiable type: ' .. kind .. '.')
  end
  return table.concat(s)
end

json.null = {}  -- This is a one-off table to represent the null value.

function json.parse(str, pos, end_delim)
  pos = pos or 1
  if pos > #str then error('Reached unexpected end of input.') end
  local pos = pos + #str:match('^%s*', pos)  -- Skip whitespace.
  local first = str:sub(pos, pos)
  if first == '{' then  -- Parse an object.
    local obj, key, delim_found = {}, true, true
    pos = pos + 1
    while true do
      key, pos = json.parse(str, pos, '}')
      if key == nil then return obj, pos end
      if not delim_found then error('Comma missing between object items.') end
      pos = skip_delim(str, pos, ':', true)  -- true -> error if missing.
      obj[key], pos = json.parse(str, pos)
      pos, delim_found = skip_delim(str, pos, ',')
    end
  elseif first == '[' then  -- Parse an array.
    local arr, val, delim_found = {}, true, true
    pos = pos + 1
    while true do
      val, pos = json.parse(str, pos, ']')
      if val == nil then return arr, pos end
      if not delim_found then error('Comma missing between array items.') end
      arr[#arr + 1] = val
      pos, delim_found = skip_delim(str, pos, ',')
    end
  elseif first == '"' then  -- Parse a string.
    return parse_str_val(str, pos + 1)
  elseif first == '-' or first:match('%d') then  -- Parse a number.
    return parse_num_val(str, pos)
  elseif first == end_delim then  -- End of an object or array.
    return nil, pos + 1
  else  -- Parse true, false, or null.
    local literals = {['true'] = true, ['false'] = false, ['null'] = json.null}
    for lit_str, lit_val in pairs(literals) do
      local lit_end = pos + #lit_str - 1
      if str:sub(pos, lit_end) == lit_str then return lit_val, lit_end + 1 end
    end
    local pos_info_str = 'position ' .. pos .. ': ' .. str:sub(pos, pos + 10)
    error('Invalid json syntax starting at ' .. pos_info_str)
  end
end

return json
```

#### Testing

Register a SIP endpoint/phone with following credentials to check if our `PostgreSQL` and `FreeSWITCH User Directory` integration via a LUA Script is successful. Open `fs_cli` for checking verbose logs and errors User: `7001` Password: `7001`
