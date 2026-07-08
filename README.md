# altair-slc-running-r-scripts-inside-a-slc-datastep
Altair slc running r scripts inside a slc datastep
    %let pgm=altair-slc-running-r-scripts-inside-a-slc-datastep;

    %stop_submission;

    Altair slc running r scripts inside a slc datastep

    Calling  r within a slc datastep and computing the area of circles and exporting and slc datastep.

    Too long to post, see github
    https://github.com/rogerjdeangelis/altair-slc-running-r-scripts-inside-a-slc-datastep

    Drop down macro on the end of this message and in
    https://github.com/rogerjdeangelis/utl-macros-used-in-many-of-rogerjdeangelis-repositories


    SOAPBOX ON

    Note: This solves some shortcommings of the slc proc r and proc python.

      1 proc r and proc python cannot be called from a macro.
      2 proc r and proc python cannot resolve macro variables.
      3 Unfortunately this makes it impossible to call a macro that contains 'proc r'
        and create a slc dataset.
        You can output a r dataframe to a feather file and use hard coded 'proc r'
        inputs and outputs to create an slc dataset. You can't even resolve
        macro variables in the hardcoded 'proc r'. You can use a dyastep eith 'call execute' to create
        'proc r code' with inputs and outputs, but this is really a kludge.
      4 It seems that dosubl is faster in the slc then sas?

    SOAPBOX OFF

    /*                   _
    (_)_ __  _ __  _   _| |_
    | | `_ \| `_ \| | | | __|
    | | | | | |_) | |_| | |_
    |_|_| |_| .__/ \__,_|\__|
            |_|
    */

    /*--- FOR CONVENIENCE PUT THIS IN YOUR AUTOEXEC ---*/

    libname workx sas7bdat "d:/wpswrkx";

    proc datasets lib=workx kill;
    run;quit;

    data workx.have;
     input radius;
    cards4;
    1
    2
    3
    4
    ;;;;
    run;


    /*******************************************************************/
    /* WORKX.HAVE total obs=4 24FEB2022:13:36:40                       */
    /*                                                                 */
    /* Obs    RADIUS                                                   */
    /*                                                                 */
    /*  1        1                                                     */
    /*  2        2                                                     */
    /*  3        3                                                     */
    /*  4        4                                                     */
    /*******************************************************************/

    %symdel radius area / nowarn;

    data workx.area_of_circle;

      set workx.have;

      call symputx('radius',radius);

      * drop down to R;
      rc=dosubl(
          '%slc_submit_r64x(
                 "
                 area<-pi * &radius * &radius;
                 print(area);
                 writeClipboard(as.character(area));
                 "
                ,return=area
                ,resolve=Y)
           ');

      area=symgetn('area');
      putlog "area  " area;

      drop rc;


    run;quit;

    proc print data=workx.area_of_circle;
    run;quit;

    /**************************************************************************************************************************/
    /*  Altair SLC                                                                                                            */
    /*                  CIRCLE                                                                                                      */
    /* Obs    RADIUS     AREA                                                                                                 */
    /*                                                                                                                        */
    /*  1        1       3.1416                                                                                               */
    /*  2        2      12.5664                                                                                               */
    /*  3        3      28.2743                                                                                               */
    /*  4        4      50.2655                                                                                               */
    /**************************************************************************************************************************/

    /*
    | | ___   __ _
    | |/ _ \ / _` |
    | | (_) | (_| |
    |_|\___/ \__, |
             |___/
    */

    Altair SLC
    LIST: 11:36:11

    Altair SLC
    > area<-pi * 1 * 1;   print(area);   writeClipboard(as.character(area));
    [1] 3.141593

    Altair SLC
    > area<-pi * 2 * 2;   print(area);   writeClipboard(as.character(area));
    [1] 12.56637

    Altair SLC
    > area<-pi * 3 * 3;   print(area);   writeClipboard(as.character(area));
    [1] 28.27433

    Altair SLC
    > area<-pi * 4 * 4;   print(area);   writeClipboard(as.character(area));
    [1] 50.26548


    /*
     _ __ ___   __ _  ___ _ __ ___
    | `_ ` _ \ / _` |/ __| `__/ _ \
    | | | | | | (_| | (__| | | (_) |
    |_| |_| |_|\__,_|\___|_|  \___/

    */

    data _null_;
       file "c:/wpsoto/slc_submit_r64x.sas";
       input;
       put _infile_;
    cards4;
    %macro slc_submit_r64x(
          pgmx
         ,return=N
         ,resolve=N
         )/des="Semi colon separated set of R commands - drop down to R";

      /*--- THIS DROP DOWN SUPPORTS THREE QUOTES, SINGLE QUOTE, DOUBLE QUOTE AND BACTIC      ---*/
      /*--- YOU CAN RESOLVE DOUBLE QUOTED MACRO VARIABLES INSIDE SINGLE QUOTES USING BACTIC  ---*/
      /*--- The macro variable inside the r program, area<-'&radius', can be resolved        ---*/
      /*--- THE NOTEPAD CLIPBOARD IS USE TO PASS MACRO CREATE DBY R BACK TO THE DATASTEP     ---*/

      %utlfkil(c:/temp/r_pgm.txt);

      * clear clipboard;
      filename _clp clipbrd;
      data _null_;
        file _clp;
        put " ";
      run;quit;

      * WRITE THE PROGRAM TO A TEMPORARY FILE AND LOG;

      filename r_pgm "c:/temp/r_pgm.txt" lrecl=32766 recfm=v;

      data _null_;
        length pgm $32756;
        file r_pgm;
        if substr(upcase("&resolve"),1,1)="Y" then do;
            pgm=resolve(&pgmx);
         end;
        else do;
            pgm=&pgmx;
         end;
         if index(pgm,"`") then cmd=tranwrd(pgm,"`","27"x);
        put pgm;
        putlog pgm;
      run;

      * PIPE FILE THROUGH R;

      filename rut pipe "C:\Progra~1\R\R-4.5.2\bin\r.exe --vanilla --quiet --no-save < c:/temp/r_pgm.txt";
      data _null_;
        file print;
        infile rut recfm=v lrecl=32756;
        input;
        put _infile_;
        putlog _infile_;
      run;

      filename rut clear;
      filename r_pgm clear;

      * USE THE CLIPBOARD TO CREATE MACRO VARIABLE;

      %if %upcase(%substr(&return.,1,1)) ne N %then %do;
        filename clp clipbrd ;
        data _null_;
         infile clp;
         input;
         putlog "macro variable &return = " _infile_;
         call symputx("&return.",_infile_,"G");
        run;quit;
      %end;

    %mend slc_submit_r64x;
    ;;;;
    run;

    /*              _
      ___ _ __   __| |
     / _ \ `_ \ / _` |
    |  __/ | | | (_| |
     \___|_| |_|\__,_|

    */
