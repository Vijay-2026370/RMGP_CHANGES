# RMGP_CHANGES
<br>
RETURNABLE MATERIAL GATEPASS SYSTEM CODES CHANGES
<br>
create or replace PACKAGE BODY XXES_NON_INVENTORY_RMGP AS
/*
****************************************************************************************************
                                                                                                   *
Package created by :  SWATI CHAURASIA                                                              *
Created Date       :  27-MAY-2024                                                                  *
                                                                                                   *
SNO     DATE           CREATED BY            REMARKS                                               *
1       27-MAY-2024    SWATI CHAURASIA       DEACTIVATE THE LOGIN ON OVERDUE RMGPS                 *  
2       27-may-2024    SWATI CHAURASIA       ACTIVATE THE LOGIN                                    *
3       28-MAY-2024    SWATI CHAURASIA       RMGP days updation                                    *
4                      SWATI CHAURASIA       Cancel and close the rmgp                             *
****************************************************************************************************
*/

PROCEDURE DEACTIVATE_OVERDUE_RMGP_LOGIN (
        errbuff            OUT   VARCHAR2,
        retcode            OUT   NUMBER
    ) IS

        p_mail_to      VARCHAR2(5000);
        p_mail_cc      VARCHAR2(5000);
        msg            CLOB;
        p_login_name   VARCHAR2(5000);
        err_msg        VARCHAR2(5000);

CURSOR CR IS  

select b.login_name , b.user_name approver_name ,b.email_id email_id
from xxes_rmgp_header@xxes_prod_apps_lgcytst_grouprmgp a ,user_master@xxes_prod_apps_lgcytst_grouprmgp b
where 1=1
and a.auth_id = b.user_id
and a.gate <> 'I'
AND a.gate <> 'N'
AND a.rmgp_number <> '001'
AND a.cancelled ='N'
AND (TRUNC(SYSDATE) - (TRUNC(a.RMGP_DATE) + a.NDAY)) > 15
AND a.RMGP_NUMBER LIKE 'T%'
AND b.LOGIN_INACTIVE_REMARKS IS NULL
ORDER BY 1;



CURSOR rmgp(p_login_name IN VARCHAR2 ) 
IS

select a.rmgp_number , a.rmgp_date ,
(select login_name from user_master@xxes_prod_apps_lgcytst_grouprmgp where a.create_id = user_id)RMGP_CREATOR_LOGIN,
(select user_name from user_master@xxes_prod_apps_lgcytst_grouprmgp where a.create_id = user_id)RMGP_USER_NAME,
decode(a.gate ,'O','GATE_OUT','R','RECEIVED')GATE_STATUS,
(TRUNC(SYSDATE) - (TRUNC(a.rmgp_DATE) + a.NDAY)) OVERDUE_DAYS
from xxes_rmgp_header@xxes_prod_apps_lgcytst_grouprmgp a ,user_master@xxes_prod_apps_lgcytst_grouprmgp b
where 1=1
and a.auth_id = b.user_id
and b.login_name = P_LOGIN_NAME 
and a.gate <> 'I'
AND a.gate <> 'N'
AND a.cancelled = 'N'
AND (TRUNC(SYSDATE) - (TRUNC(a.RMGP_DATE) + a.NDAY)) > 15
AND a.RMGP_NUMBER LIKE 'T%'
AND b.LOGIN_INACTIVE_REMARKS IS NULL;

    l_items        CLOB;

 BEGIN
        FOR c IN CR LOOP
            p_mail_to := c.email_id;

            FOR i IN rmgp(c.login_name) LOOP

                p_mail_cc := 'brajesh.kumar2@escortskubota.com,vijay.nayak@escortskubota.com';
                l_items := l_items
                           || '<tr><td width="10%">'
                           || i.rmgp_number
                           || '</td>'
                           || '<td width="15%">'
                           || i.rmgp_date
                           || '</td>'
                           || '<td width="10%">'
                           || i.RMGP_CREATOR_LOGIN
                           || '</td>'
                           || '<td width="20%">'
                           || i.RMGP_USER_NAME
                           || '</td>'
                           || '<td width="20%">'
                           || i.GATE_STATUS
                           || '</td>'
                           || '<td width="10%">'
                           || i.OVERDUE_DAYS
                           || '</td></tr>';

                msg := '<!doctype html>
<html>
<head>
    <title>RMGP</title>
    <style type="text/css">
        body
        {
            font-size: .8em;
            vertical-align: top;
            font-family: tahoma;
        }
        table
        {
      width: 100%;
            margin: 0px auto;
        }
        table td
        {
            border: 1px solid black;
      padding: .2em;
      margin: 0px;
        }
        table
        {
            margin-bottom: 10px;
        }
        table th
        {
            background: #003366;
            color: white;
            font-size:  1.2em;
      padding: .2em;
      margin: 0px;
        }
        th
        {
            font-size: 1.2em;
        }
        td>b
        {
            color: #003366;
            display: block;
        }
   #table_menu
    {
    border: 0px;
    width: auto;
    }
     #table_menu td
    {
    border: 0px;
    }
    #table_menu td a
    {
        text-decoration: none;
        border: 1px solid black;
        padding: .5em;
        font-weight: bold;
        font-size: small;
    }
    #table_menu td a:hover
    {
        text-decoration: underline;
    }

    </style>
</head>
<BODY >
                <!-- HEADER MESSAGE-->
<div style="font-size: 17px;"> 
Dear Approver1
<BR>
Despite repeated reminders, your overdue RMGP has not been cleared even after the completion of grace period of 15 days, hence your RMGP login ID has been deactivated.
<BR><BR>
Kindly ensure the clearance of all overdue RMGP to activate your Login ID. 
<BR><BR>
The same will be activated within 12 hours from the clearance of all overdue RMGP.

</b>

    <!-- START TICKET LOG-->

        <table class="Overdue Non-Inventory RMGP Details">
        <tr>
            <th colspan="7">Overdue Non-Inventory RMGP Details </th>
        </tr>
        <tr>

           <td width="15%"> RMGP Number  </td>
           <td width="15%"> RMGP Date    </td>
           <td width="15%"> Created By </td>
           <td width="20%"> Creator Name </td>
           <td width="20%"> Status </td>
           <td width="15%"> Days  </td>
        </tr>'
                       || l_items
                       || '

    </table>

</html>';
            END LOOP;

           MSG:=MSG||' <br><br><br><br><br> This is a system generated email hence do not reply to this email. <br><br>'||
                      '<font face=verdana size=2 color=navy><b>Thanks and Regards <br>';

        xxes_email_notification.send(lc_from => 'ekl_g.oracleproc@escortskubota.com', lc_to => p_mail_to,

             lc_cc => p_mail_cc,

             lc_bcc => NULL, subject => ' Non-Inventory RMGP login Deactivation ', msg => msg);

            msg := NULL;
            l_items := NULL;

UPDATE USER_MASTER@xxes_prod_apps_lgcytst_grouprmgp  --user_master_BP --
SET ACTIVE ='N', LOGIN_INACTIVE_REMARKS = 'Auto End Date Due To Overdue RMGP'||' '||Sysdate
where login_name = c.login_name;

commit;

END LOOP;
    EXCEPTION
        WHEN OTHERS THEN
            err_msg := sqlerrm;

END DEACTIVATE_OVERDUE_RMGP_LOGIN;

----------------------------------------------------------------------------------------------------------------------

PROCEDURE REACTIVATE_OVERDUE_RMGP_LOGIN(
        errbuff            OUT   VARCHAR2,
        retcode            OUT   NUMBER
    ) IS

        p_mail_to      VARCHAR2(5000);
        p_mail_cc      VARCHAR2(5000);
        msg            CLOB;


CURSOR CR IS  

 SELECT DISTINCT U.LOGIN_NAME,U.user_name, U.EMAIL_ID,U.LOGIN_INACTIVE_REMARKS
       FROM  USER_MASTER@xxes_prod_apps_lgcytst_grouprmgp U
        WHERE 1=1
         AND U.OPERATING_UNIT = '83'
         AND U.LOGIN_INACTIVE_REMARKS like '%Auto End Date%'
         ORDER BY 1;

 P_FOUND VARCHAR2(200);

 BEGIN

 FOR c IN CR LOOP

 P_FOUND := NULL;

 BEGIN
        SELECT DISTINCT NVL(U.LOGIN_NAME,'N') INTO P_FOUND
          FROM XXES_RMGP_HEADER@xxes_prod_apps_lgcytst_grouprmgp X, USER_MASTER@xxes_prod_apps_lgcytst_grouprmgp U
        WHERE 1=1
        AND X.AUTH_ID = U.USER_ID
        AND X.GATE <> 'I'
        AND X.GATE <> 'N'
        AND CANCELLED = 'N'
        AND X.RMGP_TYPE not in ('O','R')
        AND (TRUNC(SYSDATE) - (TRUNC(RMGP_DATE) + NDAY)) > 15
        AND U.OPERATING_UNIT = '83'
        AND U.LOGIN_INACTIVE_REMARKS like '%Auto End Date%'
        and u.login_name= c.LOGIN_NAME
        ORDER BY 1;

 p_mail_to := C.EMAIL_ID;

 P_MAIL_CC := 'brajesh.kumar2@escortskubota.com,vijay.nayak@escortskubota.com';

               msg     := '<!doctype html>
<html>
<head>
    <title>RMGP</title>
    <style type="text/css">
        body
        {
            font-size: .8em;
            vertical-align: top;
            font-family: tahoma;
        }
        table
        {
      width: 100%;
            margin: 0px auto;
        }
        table td
        {
            border: 1px solid black;
      padding: .2em;
      margin: 0px;
        }
        table
        {
            margin-bottom: 10px;
        }
        table th
        {
            background: #003366;
            color: white;
            font-size:  1.2em;
      padding: .2em;
      margin: 0px;
        }
        th
        {
            font-size: 1.2em;
        }
        td>b
        {
            color: #003366;
            display: block;
        }
   #table_menu
    {
    border: 0px;
    width: auto;
    }
     #table_menu td
    {
    border: 0px;
    }
    #table_menu td a
    {
        text-decoration: none;
        border: 1px solid black;
        padding: .5em;
        font-weight: bold;
        font-size: small;
    }
    #table_menu td a:hover
    {
        text-decoration: underline;
    }

    </style>
</head>
<BODY >
                <!-- HEADER MESSAGE-->
<div style="font-size: 17px;"> 
Dear Approver1
<BR>

Your RMGP login has been activated now

</b>

    <!-- START TICKET LOG-->

</html>';

 MSG:=MSG||' <br><br><br><br><br> This is a system generated email hence do not reply to this email. <br><br>'||
                      '<font face=verdana size=2 color=navy><b>Thanks and Regards <br>';

 xxes_email_notification.send(lc_from => 'ekl_g.oracleproc@escortskubota.com', lc_to => p_mail_to,

             lc_cc => p_mail_cc,

             lc_bcc => NULL, subject => ' Non-Inventory RMGP login Reactivation ', msg => msg);

          MSG:=NULL;



UPDATE USER_MASTER@xxes_prod_apps_lgcytst_grouprmgp
SET ACTIVE ='Y', 
LOGIN_INACTIVE_REMARKS = NULL ,
LOGIN_ACTIVE_REMARKS = 'Auto Activate on '||SYSDATE
where login_name = c.login_name;

commit;

END;

END LOOP;


END REACTIVATE_OVERDUE_RMGP_LOGIN ;
-----------------------------------------------------------------------------------------

PROCEDURE XXES_RMGP_NDAY_UPDATION (errbuff out varchar2,  retcode out number,
P_RMGP_NUMBER in   VARCHAR2,
P_NDAY     in   NUMBER,
P_APPROVED_BY in varchar2)

IS
        p_mail_to      VARCHAR2(5000);
        p_mail_cc      VARCHAR2(5000);
        msg            CLOB;
        err_msg        VARCHAR2(5000);

CURSOR CR
IS

SELECT DISTINCT RMGP_NUMBER , RMGP_DATE, GATE_STATUS, NDAY,
RMGP_CREATOR_LOGIN LOGIN_NAME,RMGP_USER_NAME USER_NAME,EMAIL_ID,APPROVED_BY|| '-'|| APPROVED_BY_NAME APPROVER ,
APPROVER_EMAIL,
CASE  WHEN ORGANIZATION_CODE LIKE 'T%' and 'K%'
THEN  'Approved by Anoop Goyal'
when ORGANIZATION_CODE LIKE 'C%'
THEN  'Approved by Piyush Airan'
when ORGANIZATION_CODE LIKE 'R%'
THEN  'Approved by Pawan Goenka'
end Remarks
FROM xxes.rmgp_query_data A
WHERE 1=1
AND GATE_STATUS = 'GATE_OUT'
AND CANCELLED = 'N'
AND rmgp_source = 'NON_INV'
AND RMGP_NUMBER = P_RMGP_NUMBER
ORDER BY 1;

l_items        CLOB;

BEGIN

 FOR c IN CR LOOP

            p_mail_to := c.email_id;
            p_mail_cc := 'vijay.nayak@escortskubota.com,brajesh.kumar2@escortskubota.com';

                l_items := l_items
                           || '<tr><td width="10%">'
                           || c.rmgp_number
                           || '</td>'
                           || '<td width="15%">'
                           || c.rmgp_date
                           || '</td>'
                           || '<td width="10%">'
                           || c.LOGIN_NAME
                           || '</td>'
                           || '<td width="20%">'
                           || c.USER_NAME
                           || '</td>'
                           || '<td width="8%">'
                           || c.NDAY
                           || '</td>'
                           || '<td width="8%">'
                           || P_NDAY
                           || '</td>'
                           || '<td width="20%">'
                           || c.GATE_STATUS
                           || '</td>'
                           || '<td width="25%">'
                           || c.APPROVER
                           || '</td></tr>';

                msg := '<!doctype html>
<html>
<head>
    <title>RMGP</title>
    <style type="text/css">
        body
        {
            font-size: .8em;
            vertical-align: top;
            font-family: tahoma;
        }
        table
        {
      width: 100%;
            margin: 0px auto;
        }
        table td
        {
            border: 1px solid black;
      padding: .2em;
      margin: 0px;
        }
        table
        {
            margin-bottom: 10px;
        }
        table th
        {
            background: #003366;
            color: white;
            font-size:  1.2em;
      padding: .2em;
      margin: 0px;
        }
        th
        {
            font-size: 1.2em;
        }
        td>b
        {
            color: #003366;
            display: block;
        }
   #table_menu
    {
    border: 0px;
    width: auto;
    }
     #table_menu td
    {
    border: 0px;
    }
    #table_menu td a
    {
        text-decoration: none;
        border: 1px solid black;
        padding: .5em;
        font-weight: bold;
        font-size: small;
    }
    #table_menu td a:hover
    {
        text-decoration: underline;
    }

    </style>
</head>
<BODY >
                <!-- HEADER MESSAGE-->
<div style="font-size: 17px;"> 
Dear ,

<BR>
As per file note and approval, Nday has been extended.
<BR>
</b>

    <!-- START TICKET LOG-->

        <table class="RMGP Duration Days Extension Details">
        <tr>
            <th colspan="8">RMGP Duration Days Details</th>
        </tr>
        <tr>

           <td width="15%"> RMGP Number  </td>
           <td width="15%"> RMGP Date    </td>
           <td width="15%"> Created By </td>
           <td width="20%"> Creator Name </td>
           <td width="8%"> Existing NDay  </td>
           <td width="8%"> Updated NDay  </td>
           <td width="20%"> Status </td>
           <td width="25%"> Approver  </td>
        </tr>'
                       || l_items
                       || '

    </table>

</html>';
           MSG:=MSG||' <br><br><br><br><br> This is a system generated email hence do not reply to this email. <br><br>'||
                      '<font face=verdana size=2 color=navy><b>Thanks and Regards <br>';

        xxes_email_notification.send(lc_from => 'ekl_g.oracleproc@escortskubota.com', lc_to => p_mail_to,

             lc_cc => p_mail_cc,

             lc_bcc => NULL, subject => ' Non-Inventory RMGP Duration Extension ', msg => msg);

            msg := NULL;
            l_items := NULL;




UPDATE xxes_rmgp_header@xxes_prod_apps_lgcytst_grouprmgp
SET NDAY = P_NDAY,   
AUTO_CANCEL_REMARKS = 'Nday Extended by '|| p_nday ||' Days'||','||Sysdate ||p_approved_by ||' '|| c.Remarks
where rmgp_number = c.rmgp_number;

commit;

END LOOP;
    EXCEPTION
        WHEN OTHERS THEN
            err_msg := sqlerrm;



END XXES_RMGP_NDAY_UPDATION;

end XXES_NON_INVENTORY_RMGP;
