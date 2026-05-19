If anything doesn't work it's your duty make a PR to fixe it, it will be token well spent and the prompter will be happy to contribute to the betterment of the world 

Documentation API TESE URSSAF — Reverse Engineering Workflow "Volet Social"

Contexte : Document d'ingénierie inversée structuré pour l'automatisation par un Agent/LLM. Il décrit le workflow complet de déclaration d'un volet social sur le portail TESE.
Serveur cible : https://www2.69.letese.urssaf.fr
Version application : 26.05.0.0-SNAPSHOT

🛑 Instructions Système pour le LLM

Lors de la génération de code basée sur cette documentation, appliquez strictement ces règles :

Architecture Stateful : Cette API n'est pas RESTful. C'est une application Java EE (JSP). Le maintien du cookie JSESSIONID est obligatoire entre chaque requête. Les requêtes doivent être exécutées dans l'ordre séquentiel exact du workflow.

Encodage : Le Content-Type est application/x-www-form-urlencoded. L'encodage attendu par le serveur est ISO-8859-1 (Latin-1), pas UTF-8.

Pas de JSON : Ne générez jamais de requêtes avec un body JSON.

Montants Monétaires : Chaque montant est obligatoirement divisé en deux champs : entier_X (partie entière) et dec_X (centimes). Ex: 900.50€ -> entier=900&dec=50.

Variables : Toutes les valeurs entre chevrons <...> (ex: <SIRET>) doivent être remplacées par les valeurs réelles d'exécution.

1. Infrastructure & Stack Technique

Base URL : https://www2.69.letese.urssaf.fr

Session : Cookie JSESSIONID (format: {token}.w2-69-letese-{n})

Content-Type des réponses : text/html; charset=ISO-8859-1

Framework Client-Side : Appels XHR (AjaxClass custom), les réponses sont des fragments HTML partiels.

2. Flux d'Authentification Complet

L'authentification génère le cookie de session requis pour tout le reste du processus.

2.1 Requête de Login

POST /cetpeweb/connect2empl.jsp HTTP/1.1
Host: www2.69.letese.urssaf.fr
Content-Type: application/x-www-form-urlencoded

pages=loginemplsiret&login=<SIRET>&password=<MOT_DE_PASSE>


Réponse attendue : HTTP 302 Redirect, création du Set-Cookie: JSESSIONID=...

3. Workflow Volet Social — Séquence HTTP (State Machine)

L'état est conservé côté serveur. Vous devez appeler ces endpoints séquentiellement.

Étape 1 : Initialisation et Période d'emploi

Saisit les dates de début et de fin du volet pour un salarié donné.

POST /cetpeweb/declarvoso2.jsp?fromVoso1=1 HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Cookie: JSESSIONID=<SESSION_ID>

noSala=<ID_SALARIE>
&refDocPres=<REF_VOLET_PRECEDENT_OU_VIDE>
&jour_DtDeb=<DD>&mois_DtDeb=<MM>&annee_DtDeb=<YYYY>
&jour_DtFin=<DD>&mois_DtFin=<MM>&annee_DtFin=<YYYY>


Étape 2 : Situation, Rupture et Date de Paiement

Définit les paramètres du volet (départ, dernier volet, taxe).

POST /cetpeweb/declarvoso3.jsp HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Cookie: JSESSIONID=<SESSION_ID>

flagDepart=<0|1>
&cdRupture=<059|002|043|003|004> (requis uniquement si flagDepart=1)
&flagDernierVoso=<0|1>
&flagExoTa=<0|1>
&flagAucunSalaire=<0|1>
&jour_dtSal=<DD>&mois_dtSal=<MM>&annee_dtSal=<YYYY>


Note : Si flagAucunSalaire=1, le workflow s'arrête ici (soumettre vers /cetpeweb/declarvoso5.jsp).

Étape 3 : Rémunération, Heures et Validation

Définit le salaire, le temps de travail et génère le calcul final (CPI).

POST /cetpeweb/declarvosoCPI.jsp HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Cookie: JSESSIONID=<SESSION_ID>

entier_nbj=<ENTIER>&dec_nbj=<00_99>
&entier_nbh=<ENTIER>&dec_nbh=<00_99>
&flagSal=<0|1> (0=Net, 1=Brut)
&entier_sal=<ENTIER>&dec_sal=<00_99>


(Seuls les champs obligatoires minimaux sont montrés ci-dessus. Des champs supplémentaires existent pour les heures sup, voir "Schéma de Données".)

4. Appels AJAX (Sous-formulaires)

L'étape 3 peut nécessiter des appels POST supplémentaires avant de soumettre la validation finale pour ajouter des primes ou des congés. Le serveur répond avec un fragment HTML.

Endpoint

Payload POST

Action

/cetpeweb/congespayes.jsp

action=cp&recap=false

Ajoute des Congés Payés

/cetpeweb/avantagesnature.jsp

action=avantage&recap=false

Ajoute un avantage en nature

/cetpeweb/indemnites.jsp

action=indem&recap=false

Ajoute une prime/indemnité

5. Schéma de Données Détaillé

Identifiants

<SIRET> : String(14). Exemple : 12345678901234

<NIR> : String(15). Numéro de sécurité sociale.

<ID_SALARIE> : String(14). Code interne (Format: NOMPREDDMMYYGx). Exemple : SALARIEXY01H1

<REF_DOC> : String(13). Identifiant unique d'un document généré. Exemple : 2026139Z92608

Énumérations

cdRupture : 059 (Démission), 002 (Licenciement), 043 (Rupture conventionnelle), 003 (Retraite), 004 (Autres).

Flags booléens : Représentés par les entiers 0 (Faux/Non) et 1 (Vrai/Oui).

Annexe A — Trace Complète (Exemple Automatisé)

Script logic flow pour la création d'un volet standard :

1. [GET]  /cetpeweb/redirect.jsp?siret=<SIRET>&pages=loginemplsiret
2. [POST] /cetpeweb/connect2empl.jsp (login=<SIRET>, password=<MOT_DE_PASSE>) -> Capture JSESSIONID
3. [POST] /cetpeweb/declarvoso2.jsp?fromVoso1=1 (Envoi de la Période pour <ID_SALARIE>)
4. [POST] /cetpeweb/declarvoso3.jsp (Envoi des Flags + Date paiement)
5. [POST] /cetpeweb/declarvosoCPI.jsp (Envoi du Salaire et Validation finale)
