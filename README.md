Utilizzo della CNS
==================

Per poter utilizzare la CNS in fase di autenticazione, è necessario
configurare correttamente il server web che si occuperà della maggior
parte dei processi di verifica dell'utente.

Configurazione di Apache
------------------------

La configurazione di Apache richiede che il sito sia raggiungibile
tramite https. Per fare questo si procede con la consueta configurazione
che è (dovrebbe essere) disponibile con un'installazione standard di 
Apache su Ubuntu (o altri *nix based).

Nella fattispecie:
```
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin webmaster@localhost
                DocumentRoot /var/www/html
                SSLEngine on
                SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
                SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
                SSLProtocol +TLSv1.1 +TLSv1.2
                SSLOptions +ExportCertData +StdEnvVars
                SSLVerifyClient optional
        </VirtualHost>
</IfModule>
```

L'ideale sarebbe utilizzare certificati validi (se ne possono ottenere
di gratuiti qui: https://letsencrypt.org/)

Una volta riavviato il server Apache dovrebbe essere possibile accedervi
tramite https.

Autenticazione del client tramite CNS
-------------------------------------

Ho trovato molto utile, per capire come effettuare l'autenticazione del
client, una [presentazione](http://www.slideshare.net/superg81/autenticazione-con-carta-nazionale-dei-servizi) di Gianni Forlastro e Francesco Pirrone.

Nella pratica, si scarica lo [script](https://gist.github.com/3v1n0/e371f58162795e0635f2) a cui fanno riferimento nella slide 21 e lo si esegue. 

Lo script scarica un file xml contenente i certificati relativi alle CA che emettono
le CNS. Il file fornito dal CNIPA è a [questo](https://applicazioni.cnipa.gov.it/TSL/IT_TSL_CNS.xml) indirizzo.

Per ogni TrustServiceProvider riconosciuto dall'Agenzia per l'Italia Digitale,
viene scaricato il relativo X509Certificate e salvato in formato .pem.

Alla fine dell'esecuzione, ci si dovrebbe ritrovare una novantina di file 
dei certificati. Come esposto nelle slide sopracitate, si provvede ad 
inserire tutti questi certicati in un unico file, che chiameremo
ca-CNS-bundle.crt, e posizioneremo nella cartella di apache:

```
cat *.pem >> ca-CNS-bundle.crt
mv ca-CNS-bundle.crt /etc/apache2/pki/ca-CNS-bundle.crt
```

Bisognerà ora dire ad Apache dove trovare i file dei certificati delle CA
e di forzare l'autenticazione del client in una ben definita Location:

```
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin webmaster@localhost
                DocumentRoot /var/www/html
                ...
                SSLCACertificateFile /etc/apache2/pki/ca-CNS-bundle.crt
                ...
                <Location "/cns-verify">
                        SSLVerifyClient require
                        SSLVerifyDepth  10
                </Location>
```

Per ulteriori informazioni su SSLVerify, si rimanda al sito di [Apache](https://httpd.apache.org/docs/2.4/mod/mod_ssl.html#sslverifyclient).

Riavviato Apache, dovremmo esserci.

Integrazione con PHP
--------------------

Quando, con il browser, ci troviamo sulla Location per la quale Apache
forza la verifica del client, ad esempio in https://localhost/cns-verify,
al browser viene richiesto di autenticarsi fornendo credenziali adeguate.

Per gli utenti Linux/Ubuntu, suggerisco di dare uno sguardo a [questa](http://wiki.ubuntu-it.org/Hardware/Periferiche/TesseraSanitaria) 
pagina, per effettuare la configurazione del browser per l'utilizzo della
CNS.

Se tutto è stato configurato correttamente, una volta effettuata la
verifica del client, PHP si ritrova a disposizione una serie di informazioni
all'interno della variabile ```$_SERVER```. Nella fattispecie, ecco cosa
ho a disposizione con la mia CNS:

```
["SSL_CLIENT_S_DN_C"]=> string(2) "IT" 
["SSL_CLIENT_S_DN_O"]=> string(15) "ArubaPEC S.p.A." 
["SSL_CLIENT_S_DN_OU"]=> string(31) "REGIONE AUTONOMA DELLA SARDEGNA" 
["SSL_CLIENT_S_DN_CN"]=> string(62) "AAAAAA99A99B354Z/1234567891123456.OcjosidjOIUjhohdcaalkKgUgB4=" 
["SSL_CLIENT_S_DN_G"]=> string(11) "MARIO" 
["SSL_CLIENT_S_DN_S"]=> string(6) "ROSSI" 
["SSL_CLIENT_I_DN_C"]=> string(2) "IT" 
["SSL_CLIENT_I_DN_O"]=> string(15) "ArubaPEC S.p.A." 
["SSL_CLIENT_I_DN_OU"]=> string(25) "Servizi di Certificazione" 
["SSL_CLIENT_I_DN_CN"]=> string(46) "Regione Autonoma della Sardegna - CA Cittadini" 
["SSL_CLIENT_V_START"]=> string(24) "Feb 18 12:53:58 2011 GMT" 
["SSL_CLIENT_V_END"]=> string(24) "Feb 18 12:52:32 2017 GMT" 
["SSL_CLIENT_V_REMAIN"]=> string(3) "292" 
["SSL_CLIENT_VERIFY"]=> string(7) "SUCCESS" 
["SSL_CLIENT_S_DN"]=> string(148) "SN=ROSSI,GN=MARIO,CN=AAAAAA99A99B354Z/1234567891123456.OcjosidjOIUjhohdcaalkKgUgB4=,OU=REGIONE AUTONOMA DELLA SARDEGNA,O=ArubaPEC S.p.A.,C=IT" 
["SSL_CLIENT_I_DN"]=> string(101) "CN=Regione Autonoma della Sardegna - CA Cittadini,OU=Servizi di Certificazione,O=ArubaPEC S.p.A.,C=IT" 
["SSL_CLIENT_CERT_RFC4523_CEA"]=> string(147) "{ serialNumber 123456, issuer rdnSequence:"CN=Regione Autonoma della Sardegna - CA Cittadini,OU=Servizi di Certificazione,O=ArubaPEC S.p.A.,C=IT" }"  
```

In ```$_SERVER['_SSL_CLIENT_S_DN_CN']``` abbiamo il codice fiscale dell'utente
e il codice identificativo del certificato. In ```$_SERVER['SSL_CLIENT_VERIFY']``` 
abbiamo il risultato dell'autenticazione.

Con questi parametri è possibile autenticare l'utente.

Integrazione con Symfony
------------------------

Symfony mette a disposizione l'autenticazione tramite certificati x509.
C'è un apposito [capitolo del cookbook](http://symfony.com/doc/current/cookbook/security/pre_authenticated.html) che tratta di questo aspetto.

In pratica, bisogna aggiungere, in ```app/config/security.yml```, un nuovo
provider e associarlo al firewall:
```
security:
    providers:
        cns_user_provider:
            id: app.cns_user_provider

    ...
    firewalls:
        ...
        main:
            pattern: ^/
            x509:
                provider: cns_user_provider
                user: 'SSL_CLIENT_S_DN_CN'
```

La direttiva ```user``` relativa a x509, fa si che al nostro provider
venga fornito il codice fiscale e l'identificativo del certificato
come parametro (si veda più avanti). Di default, verrebbe fornito
il campo ```SSL_CLIENT_S_DN_Email``` che invece non è gestito nella CNS.

A questo punto bisogna definire un nuovo servizio per fornire l'utente
a Symfony:

```
# in app/config/services.yml
services:
    app.cns_user_provider:
        class: AppBundle\Security\CNSUserProvider
        arguments: ['@doctrine.orm.entity_manager']
```

Lo user provider che ho definito richiede che sia fornito doctrine per poter recuperare
l'utente. Ovviamente è possibile usare qualsiasi altro sistema.

Si definisce lo [user provider](http://symfony.com/doc/current/cookbook/security/custom_provider.html) 
la cui parte saliente è il recupero dell'utente:

```
    public function loadUserByUsername($username)
    {
        // Il formato di username è, come visto prima, CODICEFISCALE/IDENTIFICATIVOCERTIFICATO 
        list($cf, $hash) = explode('/', $username);
        // Cerco l'utente a cui è associato il codice fiscale...
        $utente = $this->em->getRepository('AppBundle:Utente')->findOneBy(array(
            'codice_fiscale' => strtoupper($cf)
        ));
        if(! $utente) {
            throw new UsernameNotFoundException("Il codice fiscale non è associato a nessun utente");
        }
        return $utente;
    }
```

Regolazione delle rotte
-----------------------

Nella configurazione di Apache, abbiamo forzato l'autenticazione in una 
specifica Location mentra, per il resto delle pagine, la verifica è solo
ozionale. Questo perché altrimenti il browser darebbe errore in tutte 
le pagine a meno che sia presente la CNS (ad esempio qui: https://telematicisc.agenziaentrate.gov.it/ClientAuth/Login).

Per poter effettura l'autenticazione, ho quindi definito una nuova rotta:

```
# in app/config/routing.yml
cns_verify:
    path: /cns-verify
    defaults: {_controller: "AppBundle:Utente:cnsCheck"}
    schemes:  [https]
```

In questo modo, quando l'utente va in /cns-verify, viene effettuata
l'autenticazione che poi viene mantenuta durante tutta la sessione (fino
a che la tessera non viene estratta).

Il controller Utente, ha il metodo cnsCheckAction, che di fatto si limita a fare un redirect
verso la homepage


Problematiche
-------------

Allo stato attuale, se l'utente va in /cns-verify senza la CNS inserita,
viene generata una pagina di errore...
