[//]: # (curl使用技巧)

```shell
curl https://book.douban.com/subject/\[27199470-27199500\]/ -o "#1.html"
```
URL
       The  URL  syntax is protocol-dependent. You'll find a detailed description in RFC 3986.

You can specify multiple URLs or parts of URLs  by  writing  part  sets
 within braces as in:

 http://site.{one,two,three}.com

or you can get sequences of alphanumeric series by using [] as in:

 ftp://ftp.numericals.com/file[1-100].txt
 ftp://ftp.numericals.com/file[001-100].txt    (with leading zeros)
  ftp://ftp.letters.com/file[a-z].txt

  Nested  sequences  are not supported, but you can use several ones next
       to each other:

 http://any.org/archive[1996-1999]/vol[1-4]/part{a,b,c}.html

 You can specify any amount of URLs on the command line.  They  will  be
 fetched in a sequential manner in the specified order.

 You  can  specify a step counter for the ranges to get every Nth number
       or letter:

        http://www.numericals.com/file[1-100:10].txt
        http://www.letters.com/file[a-z:2].txt

       If you specify URL without protocol:// prefix,  curl  will  attempt  to
       guess  what  protocol  you might want. It will then default to HTTP but
       try other protocols based on often-used host name prefixes.  For  exam-
       ple,  for  host names starting with "ftp." curl will assume you want to
       speak FTP.

       curl will do its best to use what you pass to it as a URL.  It  is  not
       trying  to  validate it as a syntactically correct URL by any means but
       is instead very liberal with what it accepts.

       curl will attempt to re-use connections for multiple file transfers, so
       that  getting many files from the same server will not do multiple con-
       nects / handshakes. This improves speed. Of course this is only done on
       files  specified  on  a  single command line and cannot be used between
       separate curl invokes.

-o, --output <file>
              Write output to <file> instead of stdout. If you are using {} or
              [] to fetch multiple documents, you can use '#'  followed  by  a
              number  in  the <file> specifier. That variable will be replaced
              with the current string for the URL being fetched. Like in:

                curl http://{one,two}.site.com -o "file_#1.txt"

              or use several variables like:

                curl http://{site,host}.host[1-5].com -o "#1_#2"

              You may use this option as many times as the number of URLs  you
              have.

              See  also  the --create-dirs option to create the local directo-
              ries dynamically. Specifying the output as '-' (a  single  dash)
              will force the output to be done to stdout.