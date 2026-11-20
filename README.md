Le Dockerfile n'est pas en cause : l'image finale installe bien libldap2 et libsasl2-2 (lignes 47-52). C'est le stage test du pipeline qui pose problème.

Pourquoi ça plante.

Le stage test ne tourne pas dans cette image. Il lance uv run … manage.py test directement dans le workspace de l'agent Jenkins (/home/jenkins/agent/workspace/…/.venv).
Le Python utilisé est celui que uv a téléchargé, en 3.13.
L'agent n'a pas la bibliothèque libldap.so.2, donc import ldap échoue avant le premier test.
Autre point : les tests tournent en Python 3.13, alors que l'image est en 3.14 (PYTHON_*_VERSION=3.14-…). Le CI ne teste donc pas ce qui part en prod.

Correctif recommandé : lancer les tests dans le stage deps du Dockerfile. Ce stage contient déjà tout ce qu'il faut :

Python 3.14 ;
libldap2-dev et libsasl2-dev ;
uv ;
le venv /opt/venv.
Si l'agent sait lancer docker build / docker run, ça donne à peu près ceci :


stage('test') {
  steps {
    withCredentials([usernamePassword(credentialsId: '<artifactory>', usernameVariable: 'ARTIFACTORY_USER',
                                      passwordVariable: 'ARTIFACTORY_PWD')]) {
      sh '''
        DOCKER_BUILDKIT=1 docker build --target deps \
          --secret id=ARTIFACTORY_USER,env=ARTIFACTORY_USER --secret id=ARTIFACTORY_PWD,env=ARTIFACTORY_PWD \
          -t inventorio-deps:$BUILD_NUMBER .
        docker run --rm -v "$PWD":/app -w /app -e UV_INDEX_URL -e UV_PROJECT_ENVIRONMENT=/opt/venv \
          inventorio-deps:$BUILD_NUMBER sh -c '
            uv sync --frozen &&
            uv run --with coverage coverage run --source=apps manage.py test &&
            uv run --with coverage coverage xml -o coverage.xml'
      '''
    }
  }
}
Dans ce docker run, il faut que la variable UV_INDEX_URL contienne l'URL Artifactory avec les identifiants. Le stage deps fait --no-dev et l'export de l'URL n'existe que dans son RUN, donc uv sync et --with coverage doivent pouvoir retélécharger depuis l'index.

Si Jenkins tourne sur Kubernetes (ce que suggère le chemin /home/jenkins/agent) : ajouter au pod template un conteneur basé sur l'image python:3.14-bookworm du miroir interne. Le stage y installerait d'abord build-essential libldap2-dev libsasl2-dev, avec le même deploy/apt/ que le Dockerfile, puis lancerait les mêmes commandes uv, avec --python 3.14.

Solution de dépannage : installer libldap2 (ou libldap-2.5-0) et libsasl2-2 sur l'agent. Ça réglerait l'erreur, mais les tests resteraient en 3.13.

Si tu m'envoies le Jenkinsfile, je te donne la modification exacte du stage test.



