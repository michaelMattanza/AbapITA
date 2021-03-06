**Interfaccia**</br>
Nell'interfaccia vengono dichiarate le variabili di passaggio tra il report e il modulo. Qui, le variabili possono essere ulteriormente lavorate (meglio di no) per passare dati diversi da quelli provenienti dal report.

**Modulo** </br>
Il modulo della stampa PDF contiene il layout grafico che consente, tramite moduli e oggetti, di creare un documento organizzato con le variabili ricevute dall'interfaccia. Le variabili vengono connesse quindi agli oggetti attraverso il binding. 
Nel modulo è possibile inserire degli script in javascript e formcalc (fortemente sconsigliati, funzionano male e sono di difficile gestione). Nella sezione di passaggio variabili dall'interfaccia al modulo possono essere disattivate le variabili non utilizzate. E' inoltre possibile utilizzare "oggetti" specifici (tipo l'oggetto indirizzo) per delle formattazioni specifiche di determinate informazioni
- Logo
  - https://blogs.sap.com/2014/06/09/how-to-place-an-se78-image-on-an-adobe-form/
  
Quando si aggiunge il logo inserire nel tipo MIME <i>'IMAGE/BMP'<i/>.
  
Lo script più utilizzato è quello che permette di nascondere i campi se non sono valorizzati. Posizionarsi sul campo da condizionare e impostare la voce dello script su <i>Initialize</i> con linguaggio <i>FormCalc</i> eseguito su <i>Client:</i>

Nel caso la stampa non parta ma non venga creato un dump allora seguire i passi nel link sottostante per debuggare la stampa:
 - https://blogs.sap.com/2017/11/29/usage-and-system-error-in-sap-adobe-forms/

**Esempi di script**</br>
Gli script FormCalc vengono eseguiti solitamente sull'evento *Initialize* eseguito su *Server*.</br>
Gli script Javascript qui riportati vengono invece usati sull'evento *Form:ready* eseguito su *Client*.
FormCalc
```FormCalc
if ($ eq null) then
$.presence = "hidden"
endif
```

Javascript
```Javascript
if( this.rawValue == "" ){
	this.presence = "hidden";
}
```

Nel caso si voglia impostare la condizione sul modulo contenente il dato:

FormCalc
```FormCalc
/*Scirpt sul campo*/
if ($ eq null) then
$.parent.presence = "hidden"
endif

/*Scirpt sul modulo padre*/
if($.ZZFLAG == "X") then
	$.presence = "hidden"
endif
```

Javascript
```Javascript
/*Script sul campo*/
if( this.rawValue == "" ){
	zzmodule.presence = "hidden";
}
```
    
**Report**</br>
Nel report viene inserito il codice di estrazione e lavorazione dei dati. Una volta lavorati i dati vengono chiamate le function di open e close job.

 ```abap
  DATA: lv_fm_name   TYPE rs38l_fnam,
        ls_outpar    TYPE sfpoutputparams,
        lv_xstring   TYPE xstring.
        
 " Ottengo il logo per l'etichetta
    CALL METHOD cl_ssf_xsf_utilities=>get_bds_graphic_as_bmp
      EXPORTING
        p_object       = 'GRAPHICS'
        p_name         = 'ZF_HEADER'
        p_id           = 'BMAP'
        p_btype        = 'BCOL'
      RECEIVING
        p_bmp          = lv_xstring
      EXCEPTIONS
        not_found      = 1
        internal_error = 2
        OTHERS         = 3.
        
    " Provo a leggere il nome del function module
    TRY .
        CALL FUNCTION 'FP_FUNCTION_MODULE_NAME'
          EXPORTING
            i_name     = 'ZFM_NOME'
          IMPORTING
            e_funcname = lv_fm_name.

    ENDTRY.

    CALL FUNCTION 'FP_JOB_OPEN'
      CHANGING
        ie_outputparams = ls_outpar
      EXCEPTIONS
        cancel          = 1
        usage_error     = 2
        system_error    = 3
        internal_error  = 4
        OTHERS          = 5.

    CALL FUNCTION lv_fm_name
      EXPORTING
        is_printdata   = ls_printdata " Struttura che passo
        ix_logo        = lv_xstring " Logo che passo
      EXCEPTIONS
        usage_error    = 1
        system_error   = 2
        internal_error = 3
        OTHERS         = 4.


    CALL FUNCTION 'FP_JOB_CLOSE'
      EXCEPTIONS
        usage_error    = 1
        system_error   = 2
        internal_error = 3
        OTHERS         = 4.
 ```
 
La stampa viene lanciata con un messaggio specifico. I vari dati di questo messaggio sono contenuti nella struttura <i>NAST</i>.


**Mandare una stampa tramite mail ( n. allegati pdf )**

 ```abap
 DATA:
    lv_fm_name      TYPE rs38l_fnam,
    ls_output       TYPE fpformoutput,
    lo_pdf_content  TYPE solix_tab,

    " Gestione mail
    lo_send_request TYPE REF TO cl_bcs,
    lo_document     TYPE REF TO cl_document_bcs,
    lo_recipient    TYPE REF TO if_recipient_bcs,

    lv_sent_to_all  TYPE os_boolean,
    lv_image_xstring TYPE xstring,
    lx_document_bcs TYPE REF TO cx_document_bcs VALUE IS INITIAL.
    
    DATA(ls_outpar) = VALUE sfpoutputparams( nopreview = 'X' getpdf = 'X' nodialog  = 'X' dest = 'LP01' ).

    CALL FUNCTION 'FP_JOB_OPEN'
      CHANGING
        ie_outputparams = ls_outpar
      EXCEPTIONS
        cancel          = 1
        usage_error     = 2
        system_error    = 3
        internal_error  = 4
        OTHERS          = 5.

    CALL FUNCTION lv_fm_name
      EXPORTING
        is_printdata       = ls_print_data
      IMPORTING
        /1bcdwb/formoutput = ls_output
      EXCEPTIONS
        usage_error        = 1
        system_error       = 2
        internal_error     = 3
        OTHERS             = 4.


    TRY.
        " Allego stampa
        lo_send_request = cl_bcs=>create_persistent( ).
        lo_pdf_content = cl_bcs_convert=>xstring_to_solix( iv_xstring = ls_output-pdf ).

        DATA(lv_size) = CONV so_obj_len( xstrlen( ls_output-pdf ) ).

        lo_document = cl_document_bcs=>create_document(
                        i_type    = CONV so_obj_tp('PDF')
                        i_hex    = lo_pdf_content
                        i_length = lv_size
                        i_subject = CONV so_obj_des( |Lista Entrato Giornaliero| )
                   ).

        lo_send_request->set_document( lo_document ).
        lo_recipient = cl_cam_address_bcs=>create_internet_address(
             i_address_string = |{ <fs_partner>-smtp_address }|
        ).
        lo_send_request->add_recipient( i_recipient = lo_recipient ).

        lv_sent_to_all = lo_send_request->send(
          i_with_error_screen = 'X'
         ).

        COMMIT WORK.

      CATCH cx_root INTO DATA(lx_root_exception).

        MESSAGE ID   sy-msgid
              TYPE   sy-msgty
              NUMBER sy-msgno
              WITH   sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.

    ENDTRY.

    CALL FUNCTION 'FP_JOB_CLOSE'
      EXCEPTIONS
        usage_error    = 1
        system_error   = 2
        internal_error = 3
        OTHERS         = 4.
 ```
