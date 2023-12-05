# exifax-testfax
Glue to allow connecting HylaFax to exim 4.96, through exifax. It was written originally to assist my brother, a SysAdmin for a local community services non-profit.

It consists of this README and one file needed only long enough to initialize a `mysql` (or `mariadb`) database, so that it can be used to "detaint" fax e-mails of the form:
```
fax@12366048899.fax
```
converting them to an address of the form:
```
fax@12366048899
```
to be forwarded to Hylafax.

## How to Use -- `mysql` side

1. `git clone` this repository into some arbitrary location
2. customize `testFaxFunc.mysql` (see sections for **North America** vs. **Everywhere Else** below)
3. you may want to create a branch local to your environment with your changes so that you can track them
4. execute `testFaxFunc.mysql` against a specific database in your `msyql` instance

After this, the existence or not of `testFaxFunc.mysql` is irrelevant to the operation of fax forwarding.

Take note of the access details (username, password, database name) for where `testFaxFunc.mysql` was executed so that you can configure `exim` to access it, too.

### Customizing for North America
The default filter accepts only 10 and 11 digit numbers as is commonly required in North American contexts. Telephone numbers of any form other than `1NXXNXXXXXX` or `NXXNXXXXXX` (where N is any digit 2 through 9, and X is any digit 0 through 9) are rejected. `900` numbers which are all used to access high-cost-per-minute services are also rejected.

Additionally, the `mysql` database initializes empty tables of `allowed_codes` and `blocked_codes`.

**NB**: `blocked_codes` ONLY affects the validation of area codes if the `allowed_codes` table is empty.

Put forbidden 3-digit area codes as strings into the `blocked_codes` table. **OR**
Put authorized 3-digit area codes as strings into the `allowed_codes` table. **BUT. NOT. BOTH.**

After changes to `exim4` config files, you will be ready to forward faxes from your exim MTA.

Note: There are areas of the North American Numbering Plan Area (NANPA) that don't require 10-digit dialing but I don't know that any of these would prohibit 11 digit dialing back into the same area code. If they do, or if your contact list features 7 and 8-digit dialing cases, changes to this code are left (for the moment) as an exercise to the end-user -- and patches will be accepted.

### Customizing for Everywhere Else

In comments, after the North American implementation, there is a simple implementation that does nothing more than look for `fax@` + some arbitrary phone number (maximum size is 50 but changing that if needed is no trouble) + `.fax` and if it finds it, to return `fax@` + some arbitrary phone number. Following that there is another commented sample that might function as a starting point for a solution in the UK, or in any country whose phone system works in much the same way.

Ideas that may govern your choices (in, for instance, the UK):
* you may want to allow ALL numbers that do not start with 0 which denotes local calls (possible regex: `[1-9][0-9]+`)
* you may want to allow ALL numbers that start with a single 0 which denotes in-country non-local calls when followed by a non-zero number followed by any numbers for some length (possible regex: `0[1-9][0-9]+`)
* you may want to block ALL numbers that start with two 0s which denotes international dialing (possible regex: `00[0-9]+`)
* or maybe you will want to enable international dialing to a limited number of countries, e.g. within the EU, allowing prefixes of 00 and any of the country codes in the EU (In that case, you may want to change the definition of the `allowed_codes` and `blocked_codes` tables, too; or for Europe, since all country-codes in Europe start with 3 or 4, possible regex: `00[34][1-9][0-9]+`)

But I know just enough about different countries' telephone systems to do no more than propose possibilities here.

### Further ideas

* You may want to permit faxes to be sent only to certain numbers.
* Or you may want to permit faxes sent to certain numbers that don't satisfy other criteria.
* Or you may want to block sending faxes to certain numbers even if they satisfy the other criteria.

In those cases, you can add whitelist or blacklist tables and the associated logic. Whatever can be written into a `mysql` `FUNCTION` is available as an option.

## How to test

Once you have created the `mysql` database and applied your version of `testFaxFunc.mysql` to it, you can test the result from a `mysql` prompt with successive calls to
```
select testFax('fax@12366048899.fax');
```
replacing `12366048899` with the different test numbers you need to accept or block. Accepted addresses will be given as `fax@12366048899'. Blocked numbers will return an empty string.

## How to use -- exim side

You will need to manage database name and appropriate secrets on your system as you see fit. For more information on how to do this, [see exim's documentation](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-file_and_database_lookups.html#SECID72). Once `exim4` can use `mysql`, add this block to your routers:

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

## Notes

* in every instance where `mysql` occurs above, `mariadb` could be used just as easily. It should be noted that while `mysql` has the brand recognition (almost to the "Kleenex" point, frankly) that package no longer respects your freedom, nor the freely donated expertise of its developer community, and `mariadb` should, in every instance, be preferred. I used `mysql` in the above without intending to slight `mariadb` at all.
* the number `+1.236.604.8899` is, to my knowledge unassigned and unassignable since `236` is an overlay area code over `604` (and `250`). If this number is ever assigned to anyone, I apologize in advance to the poor sod who gets spam calls because of me.
