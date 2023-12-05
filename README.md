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

## How to Use -- mysql side

1. `git clone` this repository into some arbitrary location
2. customize `testFaxFunc.mysql`. (see sectinons for **North America** vs. **Everywhere Else** below)
3. You may want to create a branch local to your environment with your changes so that you can track them.
4. Execute `testFaxFunc.mysql` against a specific database name in your msyql instance.

After this, the existence or not of `testFaxFunc.mysql` is irrelevant to the operation of fax forwarding.

Take note of the access details for where `testFaxFunc.mysql` was executed so that you can configure `exim` to access it, too.

### Customizing for North America
The default content of the file purifies 10 and 11 digit numbers for use in North American contexts. Telephone numbers of any other form than `1NXXNXXXXXX` or `NXXNXXXXXX` (where N is any digit 2 through 9, and X is any digit 0 through 9) are rejected. `900` numbers are also rejected.

Additionally, the mysql database initializes empty tables of `allowed_codes` and `blocked_codes`.

**NB**: `blocked_codes` ONLY affects the validation of area codes if the `allowed_codes` table is empty.

Put forbidden 3-digit area codes as strings into the `blocked_codes` table. **OR**
Put authorized 3-digit area codes as strings into the `allowed_codes` table. **BUT. NOT. BOTH.**

After changes to `exim4` config files, you will be ready to forward faxes from your exim MTA.

### Customizing for Everywhere Else

In comments, after the North American implementation, there is a simple implementation that does nothing more than look for `fax@` + some arbitrary phone number (maximum size is 50 but changing that if needed is no trouble) + `.fax` and if it finds it, to return `fax@` + some arbitrary phone number.

Ideas that may govern your choices:
* You may want to allow ALL numbers that do not start with 0 (denotes local calls in, for instance the UK and much of Europe). (possible regex: `[1-9][0-9]+`)
* You may want to allow ALL numbers that start with a single 0 (denotes in-country non-local calls in UK, Europe) then a non-zero number followed by any numbers for any length. (possible regex: `0[1-9][0-9]+`)
* You may want to block ALL numbers that start with two 0s (denotes international dialing in UK, Europe). (possible regex: `00[0-9]+`)
* Or maybe you will want to enable international dialing to a limited number of countries, e.g. within the EU, allowing prefixes of 00 and any of the country codes in the EU (In that case, you may want to change the definition of `allowed_codes` and `blocked_codes` and choose regexes accordingly)

But I know just enough about different countries' telephone systems to do no more than propose possibilities here.

## How to use -- exim side

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

