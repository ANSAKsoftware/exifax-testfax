# exifax-testfax
Glue to allow connecting HylaFax, through exifax to exim 4.96, written originally to assist my brother, a SysAdmin for a local community services non-profit.

It consists of one file needed to initialize a `mysql` database, so that it can be used to "detaint" fax e-mails of the form:
```
fax@12366048899.fax
```
converting them to an address of the form:
```
fax@12366048899
```
to be forwarded to Hylafax.

## North America
The default content of the file purifies 10 and 11 digit numbers for use in North American contexts. Telephone numbers of any other form than `1NXXNXXXXXX` or `NXXNXXXXXX` (where N is any digit 2 through 9, and X is any digit 0 through 9) are rejected. `900` numbers are also rejected.

Additionally, the mysql database initializes empty tables of `allowed_codes` and `blocked_codes`.

**NB**: `blocked_codes` ONLY affects the validation of area codes if the `allowed_codes` table is empty.

Put forbidden 3-digit area codes as strings into the `blocked_codes` table. **OR**
Put authorized 3-digit area codes as strings into the `allowed_codes` table.

After changes to `exim4` config files, you will be ready to forward faxes from your exim MTA.

## Everywhere else (or more generic)

In comments, after the North American implementation, there is a simple implementation that does nothing more than look for `fax@` + some arbitrary phone number (maximum size is 50 but changing that if needed is no trouble) + `.fax` and if it finds it, to return `fax@` + some arbitrary phone number.

## Meanwhile on the exim side

You will need to manage database name and appropriate secrets on your system as you see fit. For more information on how to do this, [see exim's documentation](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-file_and_database_lookups.html#SECID72). Once `exim4` can use MySQL, add this block to your routers:

```
exifax:
      driver = manualroute
      transport = exifax
      route_list = "*.fax"
      address_data = ${lookup mysql{select testFax('${quote_mysql:$local_part@$domain}')} }
```

and add this block to your transports:

```
exifax:
    driver = pipe
    user = faxmaster
    command = "/usr/bin/faxmail -N -T -n $address_data"
```

and after reloading your config files, your system should be Good to Go™ to forward faxes via HylaFax.

