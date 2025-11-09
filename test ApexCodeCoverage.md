Vous avez **totalement raison** ! 🎯 

L'approche via `ApexCodeCoverage` de la Tooling API est **beaucoup plus précise** et optimisée. Voici comment l'implémenter :

## Solution optimisée avec ApexCodeCoverage

### 1. Script Node.js pour identifier les tests impactés

```javascript
// findImpactedTests.js
const jsforce = require('jsforce');
const fs = require('fs');

async function getImpactedTests(modifiedClasses, modifiedTriggers) {
  const conn = new jsforce.Connection({
    instanceUrl: process.env.SF_INSTANCE_URL,
    accessToken: process.env.SF_ACCESS_TOKEN
  });

  // Récupérer les IDs des classes/triggers modifiés
  const classNames = modifiedClasses.map(c => `'${c}'`).join(',');
  const triggerNames = modifiedTriggers.map(t => `'${t}'`).join(',');
  
  let whereClause = [];
  if (classNames) whereClause.push(`Name IN (${classNames})`);
  if (triggerNames) whereClause.push(`Name IN (${triggerNames})`);
  
  const apexQuery = `
    SELECT Id, Name 
    FROM ApexClass 
    WHERE ${whereClause.join(' OR ')}
  `;
  
  const apexTriggerQuery = `
    SELECT Id, Name 
    FROM ApexTrigger 
    WHERE Name IN (${triggerNames})
  `;

  // Récupérer les IDs
  const modifiedApexIds = [];
  
  if (classNames) {
    const classResult = await conn.tooling.query(apexQuery);
    modifiedApexIds.push(...classResult.records.map(r => r.Id));
  }
  
  if (triggerNames) {
    const triggerResult = await conn.tooling.query(apexTriggerQuery);
    modifiedApexIds.push(...triggerResult.records.map(r => r.Id));
  }

  // Requête ApexCodeCoverage pour trouver les tests qui couvrent ces classes/triggers
  const coverageQuery = `
    SELECT ApexTestClass.Name, ApexTestClassId, 
           ApexClassOrTrigger.Name, ApexClassOrTriggerId,
           NumLinesCovered, NumLinesUncovered
    FROM ApexCodeCoverage
    WHERE ApexClassOrTriggerId IN ('${modifiedApexIds.join("','")}')
    AND NumLinesCovered > 0
  `;

  const coverageResult = await conn.tooling.query(coverageQuery);
  
  // Extraire les noms des classes de test uniques
  const testClasses = [...new Set(
    coverageResult.records.map(r => r.ApexTestClass.Name)
  )];

  // Statistiques de couverture
  const coverageStats = {};
  coverageResult.records.forEach(record => {
    const className = record.ApexClassOrTrigger.Name;
    const testClass = record.ApexTestClass.Name;
    
    if (!coverageStats[className]) {
      coverageStats[className] = {
        totalCovered: 0,
        totalUncovered: 0,
        testClasses: []
      };
    }
    
    coverageStats[className].totalCovered += record.NumLinesCovered;
    coverageStats[className].totalUncovered += record.NumLinesUncovered;
    if (!coverageStats[className].testClasses.includes(testClass)) {
      coverageStats[className].testClasses.push(testClass);
    }
  });

  return {
    testClasses,
    coverageStats,
    totalTests: testClasses.length
  };
}

// Exemple d'utilisation
async function main() {
  const modifiedClasses = process.argv[2] ? process.argv[2].split(',') : [];
  const modifiedTriggers = process.argv[3] ? process.argv[3].split(',') : [];
  
  console.log(`Analysing coverage for ${modifiedClasses.length} classes and ${modifiedTriggers.length} triggers...`);
  
  const result = await getImpactedTests(modifiedClasses, modifiedTriggers);
  
  console.log(`\n✅ Found ${result.totalTests} impacted test classes:`);
  console.log(result.testClasses.join('\n'));
  
  console.log('\n📊 Coverage statistics:');
  console.log(JSON.stringify(result.coverageStats, null, 2));
  
  // Sauvegarder la liste des tests
  fs.writeFileSync('impacted-tests.txt', result.testClasses.join(','));
}

main().catch(console.error);
```

### 2. Script Python alternatif

```python
# find_impacted_tests.py
import os
import json
from simple_salesforce import Salesforce

def get_impacted_tests(modified_classes, modified_triggers):
    sf = Salesforce(
        instance_url=os.environ['SF_INSTANCE_URL'],
        session_id=os.environ['SF_ACCESS_TOKEN']
    )
    
    # Récupérer les IDs des classes/triggers modifiés
    modified_apex_ids = []
    
    if modified_classes:
        class_query = f"SELECT Id, Name FROM ApexClass WHERE Name IN ({','.join([f\"'{c}'\" for c in modified_classes])})"
        class_results = sf.toolingexecute(f"query/?q={class_query}")
        modified_apex_ids.extend([r['Id'] for r in class_results['records']])
    
    if modified_triggers:
        trigger_query = f"SELECT Id, Name FROM ApexTrigger WHERE Name IN ({','.join([f\"'{t}'\" for t in modified_triggers])})"
        trigger_results = sf.toolingexecute(f"query/?q={trigger_query}")
        modified_apex_ids.extend([r['Id'] for r in trigger_results['records']])
    
    # Requête ApexCodeCoverage
    coverage_query = f"""
        SELECT ApexTestClass.Name, ApexTestClassId,
               ApexClassOrTrigger.Name, ApexClassOrTriggerId,
               NumLinesCovered, NumLinesUncovered
        FROM ApexCodeCoverage
        WHERE ApexClassOrTriggerId IN ({','.join([f"'{id}'" for id in modified_apex_ids])})
        AND NumLinesCovered > 0
    """
    
    coverage_results = sf.toolingexecute(f"query/?q={coverage_query}")
    
    # Extraire les tests uniques
    test_classes = set()
    coverage_stats = {}
    
    for record in coverage_results['records']:
        test_name = record['ApexTestClass']['Name']
        class_name = record['ApexClassOrTrigger']['Name']
        
        test_classes.add(test_name)
        
        if class_name not in coverage_stats:
            coverage_stats[class_name] = {
                'totalCovered': 0,
                'totalUncovered': 0,
                'testClasses': []
            }
        
        coverage_stats[class_name]['totalCovered'] += record['NumLinesCovered']
        coverage_stats[class_name]['totalUncovered'] += record['NumLinesUncovered']
        if test_name not in coverage_stats[class_name]['testClasses']:
            coverage_stats[class_name]['testClasses'].append(test_name)
    
    return {
        'testClasses': list(test_classes),
        'coverageStats': coverage_stats,
        'totalTests': len(test_classes)
    }

if __name__ == '__main__':
    import sys
    
    modified_classes = sys.argv[1].split(',') if len(sys.argv) > 1 else []
    modified_triggers = sys.argv[2].split(',') if len(sys.argv) > 2 else []
    
    result = get_impacted_tests(modified_classes, modified_triggers)
    
    print(f"\n✅ Found {result['totalTests']} impacted test classes:")
    for test in result['testClasses']:
        print(test)
    
    # Sauvegarder pour utilisation dans le pipeline
    with open('impacted-tests.txt', 'w') as f:
        f.write(','.join(result['testClasses']))
```

### 3. Intégration dans GitHub Actions

```yaml
name: Deploy with Optimized Tests

on: [pull_request]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Important pour git diff
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: |
          npm install -g @salesforce/cli jsforce
          sf plugins install sfdx-hardis
      
      - name: Authenticate
        run: |
          echo "${{ secrets.SFDX_AUTH_URL }}" > auth.txt
          sf org login sfdx-url --sfdx-url-file auth.txt --alias target-org
          
          # Récupérer l'access token pour les scripts
          SF_ACCESS_TOKEN=$(sf org display --target-org target-org --json | jq -r '.result.accessToken')
          SF_INSTANCE_URL=$(sf org display --target-org target-org --json | jq -r '.result.instanceUrl')
          echo "SF_ACCESS_TOKEN=$SF_ACCESS_TOKEN" >> $GITHUB_ENV
          echo "SF_INSTANCE_URL=$SF_INSTANCE_URL" >> $GITHUB_ENV
      
      - name: Generate delta package
        run: |
          sf sgd:source:delta --to HEAD --from origin/main --output .
      
      - name: Extract modified Apex classes and triggers
        id: extract
        run: |
          # Extraire les classes modifiées
          CLASSES=$(grep -A 100 '<name>ApexClass</name>' package/package.xml | \
            grep '<members>' | \
            sed 's/<members>//g' | \
            sed 's/<\/members>//g' | \
            tr '\n' ',' | \
            sed 's/,$//')
          
          # Extraire les triggers modifiés
          TRIGGERS=$(grep -A 100 '<name>ApexTrigger</name>' package/package.xml | \
            grep '<members>' | \
            sed 's/<members>//g' | \
            sed 's/<\/members>//g' | \
            tr '\n' ',' | \
            sed 's/,$//')
          
          echo "classes=$CLASSES" >> $GITHUB_OUTPUT
          echo "triggers=$TRIGGERS" >> $GITHUB_OUTPUT
          
          echo "Modified classes: $CLASSES"
          echo "Modified triggers: $TRIGGERS"
      
      - name: Find impacted test classes
        run: |
          node findImpactedTests.js "${{ steps.extract.outputs.classes }}" "${{ steps.extract.outputs.triggers }}"
      
      - name: Deploy with specific tests
        run: |
          TESTS=$(cat impacted-tests.txt)
          
          if [ -z "$TESTS" ]; then
            echo "⚠️ No test classes found, using RunLocalTests"
            sf project deploy start --manifest package/package.xml \
              --test-level RunLocalTests \
              --wait 60 \
              --dry-run
          else
            echo "✅ Running specific tests: $TESTS"
            sf project deploy start --manifest package/package.xml \
              --test-level RunSpecifiedTests \
              --tests $TESTS \
              --wait 60 \
              --dry-run
          fi
```

### 4. Script Bash complet

```bash
#!/bin/bash
# find-impacted-tests.sh

set -e

# Authentification
sf org login sfdx-url --sfdx-url-file auth.txt --alias target-org

# Récupérer les tokens
ACCESS_TOKEN=$(sf org display --target-org target-org --json | jq -r '.result.accessToken')
INSTANCE_URL=$(sf org display --target-org target-org --json | jq -r '.result.instanceUrl')

# Extraire les classes modifiées
MODIFIED_CLASSES=$(git diff --name-only HEAD~1 HEAD | \
  grep "classes/.*\.cls$" | \
  sed 's/.*\///g' | \
  sed 's/\.cls//g' | \
  tr '\n' ',' | \
  sed 's/,$//')

# Requête Tooling API pour récupérer les IDs
CLASS_IDS=$(curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$INSTANCE_URL/services/data/v60.0/tooling/query?q=SELECT+Id+FROM+ApexClass+WHERE+Name+IN+('${MODIFIED_CLASSES//,/\',\'}')" | \
  jq -r '.records[].Id' | \
  tr '\n' ',' | \
  sed 's/,$//')

# Requête ApexCodeCoverage
COVERAGE_QUERY="SELECT+ApexTestClass.Name+FROM+ApexCodeCoverage+WHERE+ApexClassOrTriggerId+IN+('${CLASS_IDS//,/\',\'}')AND+NumLinesCovered>0"

TEST_CLASSES=$(curl -s -H "Authorization: Bearer $ACCESS_TOKEN" \
  "$INSTANCE_URL/services/data/v60.0/tooling/query?q=$COVERAGE_QUERY" | \
  jq -r '.records[].ApexTestClass.Name' | \
  sort -u | \
  tr '\n' ' ')

echo "Impacted test classes: $TEST_CLASSES"

# Déployer avec les tests spécifiques
sf project deploy start --manifest package/package.xml \
  --test-level RunSpecifiedTests \
  --tests $TEST_CLASSES \
  --dry-run
```

## Avantages de cette approche

✅ **Précision maximale** : Exécute uniquement les tests qui couvrent réellement le code modifié  
✅ **Performance** : Réduit drastiquement le temps d'exécution des tests  
✅ **Fiabilité** : Basé sur les données réelles de couverture de code  
✅ **Traçabilité** : Affiche les statistiques de couverture par classe

Cette approche est effectivement **bien supérieure** aux Smart Tests de sfdx-hardis ! Vous comptez l'implémenter ?

---

Vous avez mis le doigt sur le **dilemme fondamental** ! 🎯

C'est exactement pourquoi sfdx-git-delta ne fait pas ça par défaut. Il y a un **trade-off** entre :

## Le Dilemme

### ⚡ Approche Chirurgicale (Optimisée)
✅ Rapide - uniquement les tests nécessaires  
✅ Économise du temps CI/CD  
❌ **Données de couverture potentiellement obsolètes**  
❌ **Risque de manquer des nouveaux tests**

### 🔒 Approche Complète (Sécurisée)
✅ Couverture toujours à jour  
✅ Détecte tous les problèmes  
❌ **Très long** (peut prendre 30-60+ minutes)  
❌ **Coûteux en ressources CI/CD**

## Solutions Hybrides Intelligentes

### 1. **Stratégie à Deux Niveaux** (Recommandé)

```yaml
# .github/workflows/ci-hybrid.yml
name: Hybrid Test Strategy

on: [pull_request]

jobs:
  analyze-changes:
    runs-on: ubuntu-latest
    outputs:
      has-test-changes: ${{ steps.check.outputs.has_tests }}
      has-apex-changes: ${{ steps.check.outputs.has_apex }}
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Check what changed
        id: check
        run: |
          # Vérifier si des tests ont été modifiés
          TEST_CHANGES=$(git diff --name-only origin/main...HEAD | grep -E "Test\.cls$|_Test\.cls$" || echo "")
          
          # Vérifier si du code Apex a été modifié
          APEX_CHANGES=$(git diff --name-only origin/main...HEAD | grep -E "\.cls$|\.trigger$" || echo "")
          
          if [ -n "$TEST_CHANGES" ]; then
            echo "has_tests=true" >> $GITHUB_OUTPUT
            echo "⚠️ Test classes modified - will run full test suite"
          else
            echo "has_tests=false" >> $GITHUB_OUTPUT
          fi
          
          if [ -n "$APEX_CHANGES" ]; then
            echo "has_apex=true" >> $GITHUB_OUTPUT
          else
            echo "has_apex=false" >> $GITHUB_OUTPUT
          fi

  deploy-optimized:
    needs: analyze-changes
    if: needs.analyze-changes.outputs.has-test-changes == 'false'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy with surgical tests
        run: |
          echo "🎯 Using optimized test strategy (ApexCodeCoverage)"
          # Script avec ApexCodeCoverage comme montré précédemment
          node findImpactedTests.js
          # Deploy avec tests spécifiques

  deploy-complete:
    needs: analyze-changes
    if: needs.analyze-changes.outputs.has-test-changes == 'true'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy with full test suite
        run: |
          echo "🔒 Running full test suite (test classes were modified)"
          sf hardis:project:deploy:smart --check --testlevel RunLocalTests
```

### 2. **Refresh Périodique de la Couverture**

```yaml
# .github/workflows/refresh-coverage.yml
name: Refresh Code Coverage Cache

on:
  schedule:
    # Tous les jours à 2h du matin
    - cron: '0 2 * * *'
  workflow_dispatch:  # Permet le déclenchement manuel

jobs:
  refresh-coverage:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Authenticate
        run: |
          echo "${{ secrets.SFDX_AUTH_URL }}" > auth.txt
          sf org login sfdx-url --sfdx-url-file auth.txt --alias integration-org
      
      - name: Run all tests to refresh coverage
        run: |
          echo "🔄 Refreshing code coverage data..."
          sf apex run test --test-level RunLocalTests --wait 60 --target-org integration-org
      
      - name: Export coverage data
        run: |
          # Exporter les données de couverture pour analyse
          node exportCoverageData.js
          
      - name: Cache coverage data
        uses: actions/cache@v3
        with:
          path: coverage-cache/
          key: apex-coverage-${{ github.run_number }}
```

### 3. **Détection Intelligente avec Horodatage**

```javascript
// smartTestSelector.js
const jsforce = require('jsforce');
const fs = require('fs');

async function needsFullRefresh(conn) {
  // Vérifier la date de dernière modification des tests
  const lastTestModification = await conn.tooling.query(`
    SELECT MAX(LastModifiedDate) lastMod
    FROM ApexClass
    WHERE Name LIKE '%Test%'
  `);
  
  // Vérifier la date de dernière exécution complète des tests
  const lastFullRun = fs.existsSync('.last-full-test-run') 
    ? new Date(fs.readFileSync('.last-full-test-run', 'utf8'))
    : new Date(0);
  
  const lastTestMod = new Date(lastTestModification.records[0].lastMod);
  
  // Si des tests ont été modifiés depuis la dernière exécution complète
  if (lastTestMod > lastFullRun) {
    console.log('⚠️ Test classes modified since last full run');
    return true;
  }
  
  // Vérifier si cela fait plus de 7 jours depuis la dernière exécution complète
  const daysSinceLastRun = (Date.now() - lastFullRun) / (1000 * 60 * 60 * 24);
  if (daysSinceLastRun > 7) {
    console.log('⚠️ Last full test run was more than 7 days ago');
    return true;
  }
  
  return false;
}

async function getTestStrategy(conn, modifiedClasses) {
  const needsFullTests = await needsFullRefresh(conn);
  
  if (needsFullTests) {
    return {
      strategy: 'FULL',
      testLevel: 'RunLocalTests',
      reason: 'Coverage data may be stale'
    };
  }
  
  // Approche chirurgicale
  const impactedTests = await getImpactedTestsFromCoverage(conn, modifiedClasses);
  
  if (impactedTests.length === 0) {
    console.log('⚠️ No coverage data found, falling back to full tests');
    return {
      strategy: 'FULL',
      testLevel: 'RunLocalTests',
      reason: 'No coverage data available'
    };
  }
  
  return {
    strategy: 'SURGICAL',
    testLevel: 'RunSpecifiedTests',
    tests: impactedTests,
    reason: `Found ${impactedTests.length} impacted tests from coverage data`
  };
}

async function getImpactedTestsFromCoverage(conn, modifiedClasses) {
  // Code précédent pour récupérer les tests depuis ApexCodeCoverage
  // ...
}

module.exports = { getTestStrategy, needsFullRefresh };
```

### 4. **Stratégie par Type de Branche**

```yaml
# config/.sfdx-hardis.yml
testStrategy:
  # Branches de feature : optimisé
  feature/*:
    strategy: surgical
    fallbackToFull: false
    maxAge: 7  # jours
  
  # Branches de développement : hybride
  develop:
    strategy: surgical
    fallbackToFull: true
    maxAge: 3
  
  # Branches principales : toujours complet
  main:
    strategy: full
    testLevel: RunLocalTests
  
  production:
    strategy: full
    testLevel: RunLocalTests
```

### 5. **Script Complet avec Fallback Intelligent**

```javascript
// intelligentTestRunner.js
const jsforce = require('jsforce');
const { execSync } = require('child_process');
const fs = require('fs');

async function runIntelligentTests(modifiedClasses, modifiedTriggers) {
  const conn = new jsforce.Connection({
    instanceUrl: process.env.SF_INSTANCE_URL,
    accessToken: process.env.SF_ACCESS_TOKEN
  });
  
  console.log('🧠 Analyzing test strategy...');
  
  // 1. Vérifier si des tests ont été modifiés dans cette PR
  const modifiedTests = getModifiedTestClasses();
  
  if (modifiedTests.length > 0) {
    console.log('⚠️ Test classes were modified, running full test suite');
    return runFullTests();
  }
  
  // 2. Vérifier l'âge des données de couverture
  const coverageAge = await getCoverageDataAge(conn);
  const MAX_COVERAGE_AGE_DAYS = 7;
  
  if (coverageAge > MAX_COVERAGE_AGE_DAYS) {
    console.log(`⚠️ Coverage data is ${coverageAge} days old (max: ${MAX_COVERAGE_AGE_DAYS})`);
    console.log('Running full test suite to refresh coverage...');
    await runFullTests();
    return;
  }
  
  // 3. Récupérer les tests impactés depuis ApexCodeCoverage
  const impactedTests = await getImpactedTestsFromCoverage(
    conn, 
    modifiedClasses, 
    modifiedTriggers
  );
  
  if (impactedTests.length === 0) {
    console.log('⚠️ No coverage data found for modified classes');
    console.log('Falling back to full test suite...');
    return runFullTests();
  }
  
  // 4. Vérifier si les tests impactés existent toujours
  const existingTests = await verifyTestsExist(conn, impactedTests);
  
  if (existingTests.length !== impactedTests.length) {
    console.log('⚠️ Some referenced tests no longer exist');
    console.log('Falling back to full test suite...');
    return runFullTests();
  }
  
  // 5. Exécution chirurgicale
  console.log(`✅ Running ${impactedTests.length} impacted tests (surgical approach)`);
  console.log('Tests to run:', impactedTests.join(', '));
  
  return runSpecificTests(impactedTests);
}

function getModifiedTestClasses() {
  const output = execSync('git diff --name-only origin/main...HEAD').toString();
  return output.split('\n')
    .filter(f => f.match(/Test\.cls$|_Test\.cls$/))
    .map(f => f.replace(/.*\//, '').replace('.cls', ''));
}

async function getCoverageDataAge(conn) {
  // Récupérer la date de dernière exécution des tests
  const result = await conn.tooling.query(`
    SELECT MAX(TestTimestamp) lastRun
    FROM ApexCodeCoverageAggregate
  `);
  
  if (!result.records[0].lastRun) {
    return Infinity;
  }
  
  const lastRun = new Date(result.records[0].lastRun);
  const ageInDays = (Date.now() - lastRun) / (1000 * 60 * 60 * 24);
  
  return ageInDays;
}

async function verifyTestsExist(conn, testNames) {
  const query = `
    SELECT Name 
    FROM ApexClass 
    WHERE Name IN (${testNames.map(t => `'${t}'`).join(',')})
  `;
  
  const result = await conn.tooling.query(query);
  return result.records.map(r => r.Name);
}

function runFullTests() {
  console.log('🔒 Executing full test suite...');
  execSync('sf project deploy start --manifest package/package.xml --test-level RunLocalTests --wait 60 --dry-run', 
    { stdio: 'inherit' });
  
  // Marquer la date d'exécution complète
  fs.writeFileSync('.last-full-test-run', new Date().toISOString());
}

function runSpecificTests(tests) {
  console.log('🎯 Executing surgical test suite...');
  execSync(`sf project deploy start --manifest package/package.xml --test-level RunSpecifiedTests --tests ${tests.join(' ')} --wait 60 --dry-run`,
    { stdio: 'inherit' });
}

// Export pour utilisation dans d'autres scripts
module.exports = { runIntelligentTests };
```

## Recommandation Finale

**Stratégie à 3 Niveaux** :

1. **Feature branches (PR)** : Approche chirurgicale avec fallback si doutes
2. **Integration/Develop** : Full tests une fois par jour + chirurgical pour le reste
3. **Main/Production** : Toujours full tests

```yaml
# Exemple complet
if [[ "$BRANCH" == "main" || "$BRANCH" == "production" ]]; then
  # Toujours full tests
  STRATEGY="FULL"
elif [ -n "$(git diff --name-only origin/main...HEAD | grep Test\.cls)" ]; then
  # Des tests ont été modifiés
  STRATEGY="FULL"
elif [ "$COVERAGE_AGE_DAYS" -gt 7 ]; then
  # Couverture trop ancienne
  STRATEGY="FULL"
else
  # Approche chirurgicale sécurisée
  STRATEGY="SURGICAL"
fi
```

Cette approche équilibre **performance** et **fiabilité**. Qu'en pensez-vous ?
