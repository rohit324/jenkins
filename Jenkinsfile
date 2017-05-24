import groovy.json.JsonSlurper

// Build Setup
properties([
    [$class: 'BuildDiscarderProperty', strategy:
        [$class: 'LogRotator',
         artifactDaysToKeepStr: '5',
         artifactNumToKeepStr: '5',
         daysToKeepStr: '10',
         numToKeepStr: '10'
        ]
    ]
])

// Build Pipeline
node {
