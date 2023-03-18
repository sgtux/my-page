---
title: "Como previnir SQL Injection em aplicações JAVA ?"
date: 2023-02-15T09:03:20-08:00
draft: true
---

### O que é SQL Injection ?

Durante muitos anos o SQL Injection foi e ainda é uma falha de segurança crítica muito encontrada em aplicações web, esta falha basicamente se consiste em, um usuário através de uma entrada de dados, seja por um parâmetro de formulário ou parâmetro de url, conseguir executar comandos diretamente no banco de dados.

Então vamos tomar como exemplo três formas diferêntes de implementação de um método simples de login que faz acesso ao banco de dados buscando um usuário por email e senha:

Statement:
```java
public Usuario login(String email, String senha) throws SQLException {
    var query = String.format("select * from usuario where email = '%s' and senha = '%s'", email, senha);

    Statement statement = this.connection.createStatement();
    var resultSet = statement.executeQuery(query);
    Usuario usuario = null;
    
    if(resultSet.next()) {
        usuario.setId(resultSet.getLong("id"));
        usuario.setNome(resultSet.getString("nome"));
        usuario.setSobrenome(resultSet.getString("sobrenome"));
        usuario.setEmail(resultSet.getString("email"));
        usuario.setFoto(resultSet.getString("foto"));
    }

    return usuario;
}
```
JPA:
```java
public Usuario login(String email, String senha) {
    var query = String.format("select * from usuario where email = '%s' and senha = '%s'", email, senha);
    var list = entityManager.createNativeQuery(query, Usuario.class).getResultList();
    if(list.isEmpty())
        return null;
    else
        return list.get(0);
}
```

JPA com JPQL:
```java
public Usuario login(String email, String senha) {
    var jpqlquery = String.format("select u from Usuario u where u.email = '%s' and u.senha = '%s'", email, senha);
    var list = entityManager.createQuery(jpqlquery, Usuario.class).getResultList();
    if(list.isEmpty())
        return null;
    else
        return list.get(0);
}
```

Bom, então vamos entender um poucos mais o porque deste código ser vulnerável. Vamos pegar o primeiro método, que na verdade também serve para os demais.

Supondo que o email enviado é *admin@mail.com* e a senha é *123*, o comando que será enviado para o banco de dados será:

```sql
select * from usuario where email = 'admin@mail.com' and senha = '123'
```

Agora vamos supor que o email enviado é *admin@mail.com' --* e a senha *123*, o comando enviado para banco de dados será:

```sql
select * from usuario where email = 'admin@mail.com' --' and senha = '123'
```

Então mesmo que a senha esteja incorreta, o email será encontrado no banco de dados e o login terá sucesso.

E agora o que podemos fazer para garantir que nosso código não tenha esse tipo de falha, bom primeiro lugar, muito importante:
### Nunca concatene os valores enviado por usuário em comandos que fazem acesso ao banco de dados, ao invés disso utilize consultas parametrizáveis (também conhecidas como Prepared Statement)!

Então segue os exemplos de códigos com a falha de SQL Injection resolvida:

**Solução Statement:**
```java
public Usuario login(String email, String senha) throws SQLException {
    var query = String.format("select * from usuario where email = ? and senha = ?", email, senha);

    PreparedStatement preparedStatement = this.connection.prepareStatement(query);

    // PARAMETRIZAÇÃO SEGURA
    preparedStatement.setString(1, email);
    preparedStatement.setString(2, senha);

    var resultSet = preparedStatement.executeQuery();
    Usuario usuario = null;
    
    if(resultSet.next()) {
        usuario.setId(resultSet.getLong("id"));
        usuario.setNome(resultSet.getString("nome"));
        usuario.setSobrenome(resultSet.getString("sobrenome"));
        usuario.setEmail(resultSet.getString("email"));
        usuario.setFoto(resultSet.getString("foto"));
    }

    return usuario;
}
```
**Solução JPA:**
```java
public Usuario login(String email, String senha) {
    var query = String.format("select * from usuario where email = :email and senha = :senha", email, senha);
    var list = entityManager.createNativeQuery(query, Usuario.class)
        .setParameter("email", email) // PARAMETRIZAÇÃO SEGURA
        .setParameter("senha", senha)
        .getResultList();
    if(list.isEmpty())
        return null;
    else
        return list.get(0);
}
```

**Solução JPA com JPQL:**
```java
public Usuario login(String email, String senha) {
    var jpqlquery = String.format("select u from Usuario u where u.email = :email and u.senha = :senha", email, senha);
    var list = entityManager.createQuery(jpqlquery, Usuario.class)
        .setParameter("email", email) // PARAMETRIZAÇÃO SEGURA
        .setParameter("senha", senha)
        .getResultList();
    if(list.isEmpty())
        return null;
    else
        return list.get(0);
}
```

Devido a enorme quantidade de exploração do SQL Injection no passado, com vazamento de dados, danos causados, as soluções para a mitigação desta falha, como foi visto, são muito simples. Porém, talvez devido a falta de conscientização e conhecimento, muitas aplicações ainda sofrem com esta falha.