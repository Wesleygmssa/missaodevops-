# missaodevops-

image: node

stages:
  - test
  - package
  - deploy

before_script:
  - whoami
  - pwd

test site:
  stage: test
  before_script:
    - npm install -g broken-link-checker@^0.7.8 wait-on@^2.1.0
  script:
    - cd website
    - npm install && npm run start &
    - wait-on http://localhost:3000/ --timeout 90000
    - blc --recursive --exclude-external http://localhost:3000

vulnerabilities check:
  stage: test
  script:
    - cd website && npm install && npm run scan

package site:
  stage: package
  cache:
    key: site-package
    policy: push
    paths:
      - ./website/build
  artifacts:
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_NAME"
    when: always
    expire_in: 2h20min
    paths:
      - ./website/build/missaodevops
  script:
    - cd website
    - npm install
    - npm run build

deploy to develop:
  stage: deploy
  variables:
    CNAME: develop.missaodevops.surge.sh
    GIT_STRATEGY: none
  cache:
    key: site-package
    policy: pull
  before_script:
    - npm install -g surge@^0.20.1
  script:
    - surge --project ./website/build/missaodevops --domain ${CNAME}
  after_script:
    - whoami
  environment:
    name: develop
    url: http://${CNAME}
  only:
    - develop

deploy to release:
  stage: deploy
  variables:
    CNAME: $CI_COMMIT_REF_SLUG.missaodevops.surge.sh
    GIT_STRATEGY: none
  cache:
    key: site-package
    policy: pull
  before_script:
    - npm install -g surge@^0.20.1
  script:
    - surge --project ./website/build/missaodevops --domain ${CNAME}
  environment:
    name: release/$CI_COMMIT_REF_NAME
    url: http://${CNAME}
    on_stop: turnoff
  only:
    - /^release-.*$/

turnoff:
  stage: deploy
  variables:
    CNAME: $CI_COMMIT_REF_SLUG.missaodevops.surge.sh
    GIT_STRATEGY: none
  before_script:
    - npm install -g surge@^0.20.1
  script:
    - surge teardown ${CNAME}
  when: manual
  environment:
    name: release/$CI_COMMIT_REF_NAME
    url: http://${CNAME}
    action: stop
  only:
    - /^release-.*$/

deploy to production:
  stage: deploy
  variables:
    CNAME: missaodevops.surge.sh
    GIT_STRATEGY: none
  cache:
    key: site-package
    policy: pull
  before_script:
    - npm install -g surge@^0.20.1
  script:
    - surge --project ./website/build/missaodevops --domain ${CNAME}
  after_script:
    - whoami
  environment:
    name: production
    url: http://${CNAME}
  only:
    - master
  
