
## Cette série de tuto a pour objectif de vous apprendre à développer une intégration Home Assistant acccessible sur HACS. Elle se veut progressive et didactique.

---
> Tu veux développer une intégration pour Home Assistant ou tout simplement contribuer à une intégration existante ? Cette suite d'article est faite pour toi et à pour but de t'aider à te lancer. Elle est conçu comme les articles que j'aurai aimé trouver lorsque je me suis moi-même lancé.

# Les pré-requis

Pour pouvoir utiliser ces articles il est préférable de :
1. Connaitre Home Assistant un minimum et notamment les concepts de : Appareils, Entités, Intégration. Bien qu'on va devoir définir ces termes il est quand même souhaitable de connaitre ces concepts un minimum,
2. Connaitre le langage de développement Python. Cette suite d'article n'est pas un cours de Python et suppose que vous connaissez un minimum ce langage. Sinon il existe d'excellent tutos Python en ligne et il est préférable de les suivre avant d'attaquer ces articles,
3. Le métier du developpement. Le développement est un métier, à son vocabulaire et ses règles. Des notions sont nécessaires pour comprendre ces articles, même si le niveau n'est pas très élevé. Soyez à l'aise avec les notions de sources, gestionnaire de sources (Github, git), repository, classes, méthodes, tests unitaires, attributs ou faites un tuto en ligne avant d'aborder ces articles. 
4. Un compte Github : un compte Github est nécessaire pour pouvoir forker des repos et notamment celui de Home Assistant
5. Un PC ou un Mac ou une box linux bien sur.

# Remarques préliminaires

__Open Source on vous a dit :__ il existe littéralement des milliers d'intégrations pour Home Assistant. Le meilleur conseil que je peux donner est d'aller voir leur code. Tu vas apprendre énormément en consultant ce que les autres ont faits et comment ils l'ont fait. L'aspect Open-Source est précieux et super utile lorsqu'on but sur une difficulté -> n'hésites pas.

__HACS :__ `Home Assistant Community Store` est tout simplement indispensable. Il propose des un store parallèle contenant des intégrations, des composants front, des thèmes fourni par la communauté pour la communauté. Il a l'énorme avantage d'être indépendant de l'équipe core Home Assistant ce qui vous permet d'être libre dans la réalisation de vos intégrations et notamment dans la fréquence de publication. Si vous tentez de faire une intégration intégrée à Home Assistant vous devrez soumettre votre intégration à la revue de l'équipe Core et à des délais décourageant (plusieurs mois au moment j'écris ces lignes). Pour démarrer, je ne peux conseiller de démarrer avec une intégration HACS. Une fois que vous avez une intégration de qualité, vous pouvez tenter de la proposer comme une intégration Core mais pas avant.

__Un peu de lecture :__ la doc officielle est [ici](#https://developers.home-assistant.io/docs/development_index). Elle est en Anglais, pas super à jour tout le temps, n'est pas faite pour développer une intégration HACS et n'est pas très didactique. Si tu n'y connais rien, tu n'y comprendra pas grand-chose. Par contre, s'y référer de temps en temps pour étendre le champ de sa connaissance une fois qu'on a compris les concepts est une super bonne idée. Gardes bien ça en tête !


> ![Tip](/images/tips.png?raw=true)
> Tu es prêt ? Fébrile ? Ca va aller, on te dit...


Sommaire
1. _[tuto1](/tuto1.md) :_ installer son environnement de dev,
2. _[tuto2](/tuto2.md) :_ créer une intégration et une entité simple,
3. _[tuto3](/tuto3.md) :_ interactions avec d'autres entités (évènements, états, ...),
4. _tuto4 :_ pages de configuration (configFlow, traductions)
5. _tuto5 :_ aspects avancés (tests unitaires, debugging, travailler avec les sources Home Assistant)



