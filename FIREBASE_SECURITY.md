# Firebase Security Rules

Este documento contém as regras de segurança para o Firestore do projeto RutiStack.

## Problema de Permissão

Se você está recebendo o erro:
```
FirebaseError: [code=permission-denied]: Permission denied on resource project jogorutistack.
```

Isso significa que as regras de segurança do Firestore precisam ser configuradas.

## Configuração das Regras

Acesse o [Firebase Console](https://console.firebase.google.com/project/rutistack-c75f2/firestore/rules) e configure as seguintes regras:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Regras para a coleção 'scores'
    match /scores/{scoreId} {
      // Permitir leitura pública (para o ranking)
      allow read: if true;
      
      // Permitir escrita apenas com validação
      allow create: if request.auth == null  // Permite criar sem autenticação
                    && request.resource.data.playerName is string
                    && request.resource.data.playerName.size() >= 3
                    && request.resource.data.playerName.size() <= 8
                    && request.resource.data.score is number
                    && request.resource.data.score >= 0
                    && request.resource.data.score <= 10000  // Limite máximo de score
                    && request.resource.data.timestamp is timestamp
                    && request.resource.data.keys().hasAll(['playerName', 'score', 'timestamp']);
      
      // Proibir atualizações e deleções
      allow update, delete: if false;
    }
    
    // Regras para a coleção 'config' (admin panel)
    match /config/{configId} {
      // Permitir leitura pública
      allow read: if true;
      
      // Apenas admin pode escrever
      allow write: if request.auth != null && request.auth.token.admin == true;
    }
    
    // Bloquear acesso a outras coleções
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

## Recursos de Segurança Implementados

### Anti-Hack Logging

Quando um jogador salva o score, os seguintes dados são registrados no campo `gameLog`:

- `totalBoxes`: Número total de caixas empilhadas
- `finalScore`: Score final do jogo
- `gameSpeed`: Velocidade final do guindaste
- `difficulty`: Multiplicador de dificuldade
- `gravity`: Valor da gravidade
- `playDuration`: Duração da partida em segundos
- `boxTypes`: Tipos de cada caixa usada
- `userAgent`: Navegador/dispositivo do jogador
- `screenResolution`: Resolução da tela
- `timestamp`: Data/hora exata do save

### Validações

As regras acima implementam:

1. **Limite de nome**: 3-8 caracteres
2. **Limite de score**: Máximo 10.000 pontos
3. **Campos obrigatórios**: playerName, score, timestamp
4. **Tipos de dados**: Validação de tipos
5. **Somente leitura no ranking**: Scores não podem ser modificados
6. **Proteção contra excesso de writes**: Sistema de rate limiting do Firebase

## Como Aplicar

1. Acesse: https://console.firebase.google.com/project/jogorutistack/firestore/rules
2. Cole as regras acima
3. Clique em "Publicar"
4. Aguarde alguns segundos para propagação

## Monitoramento

Para monitorar tentativas de hack:

1. Acesse: https://console.firebase.google.com/project/jogorutistack/firestore/data
2. Visualize a coleção `scores`
3. Analise o campo `gameLog` de scores suspeitos
4. Compare `playDuration` com `score` para detectar anomalias

## Notas

- Scores com `playDuration` muito curto para o score alcançado são suspeitos
- `boxTypes` deve ter comprimento igual a `totalBoxes`
- `finalScore` deve ser igual a `score`
