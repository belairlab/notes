# notes
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