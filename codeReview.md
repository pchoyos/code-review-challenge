# Code Review

## Arquitectura
1. Parece que sigue una arquitectura hexagonal, sin embargo hay algunos puntos 
a tener en cuenta. AdsServiceImpl y AdsService pertenecen a la parte del core
del negocio. AdsServiceImpl es el adaptador que implementa al puerto, por lo que
movería la implementación a la parte de domain, y no iría etiquetado como un bean 
de spring, ya que el dominio debe ser independiente.
   
````
public class AdsServiceImpl implements AdsService {

    private final AdRepository adRepository;

    public AdsServiceImpl(AdRepository adRepository) {
        this.adRepository = adRepository;
    }
````
   
2. En la parte de application, en lugar de los servicios, colocaríamos el 
controlador rest con el que nos comunicaríamos con esta api. En este caso, 
utilizando las etiquetas de Spring. Ahora, ya es nuestro controlador
el encargado de ejecutar la lógica de dominio de nuestra api.
   
3. En la parte de infrastructure, sacamos el controlador a la parte de application,
como ya hemos explicado, y dejaríamos este módulo para la configuración de todo lo necesario
para que nuestra api funcione. Creamos la clase para configurar los beans, aquí nuestro
servicio AdsService será configurado como un bean de spring.
````
@Configuration
public class BeanConfiguration {

    @Bean
    AdsService adsService (AdRepository adRepository){
        return new AdsServiceImpl(adRepository);
    }
}

````
   

De esta forma, el esqueleto del proyecto quedaría así:
- aplication
    - api
        - AdsController.java
    - response
        - PublicAd.java
        - QualityAd.java
    
- domain
    - repository
        - AdRepository.java
    - service
        - AdsService.java
        - AdsServiceImpl.java
    - Ad.java
    - Constants.java
    - Picture.java
    - Quality.java
    - Typology.java
    
- infrastructure
    - config
        - BeanConfiguration.java
    - persistence
        - AdVO.java
        - PictureVO.java
        - InMemoryPersistence.java
    

## Clean Code
###Convenciones en nombre de clases:
- Modificaría AdsService y AdsServiceImpl por AdService y AdServiceImpl. 
  Aunque hay varias convenciones al respecto, la más utilizada pasa por usar nombres
  singulares de atributos. Lo mismo con la clase AdsController -> AdController.
  
###Claridad en el nombre de métodos y clases
- Modificaría la clase Constants, puesto que no me parece que se entienda su uso con ese nombre.
  Como se utiliza para el cálculo del score, la llamaría scoreConstants, de forma que así
  se entiende para qué se han definido. Por otro lado, añadiría la palabra "POINTS" a las 
  constantes, para hacer claro lo que significa su valor.
  
- Uso de palabras hardcodeadas en AdsServiceImpl#calculateScore#111-115. Podríamos crear
una lista de palabras clave a buscar dentro de la clase constant. De esta forma, si tenemos 
  que añadir una nueva palabra al cálculo, bastaría con añadirla a la lista. El cálculo quedaría así:
   ````
  int occurrences = wds.stream()
                    .distinct()
                    .filter(listOfKeyWords::contains)
                    .collect(Collectors.toSet()).size();
  score = occurrences > 0 ? score + occurrences * Constants.FIVE: score;
  ```` 
  de esta forma, si añadimos una nueva palabra clave, no tendríamos que cambiar el código.

- Por otro lado, AdController.java#calculateScore se trata de un GET, pero la realidad es que es un 
patch sobre los anuncios, tras haber calculado el score de ellos. Modificaría ese método para que fuera
  un patch, que devuelva un 204 en caso de que el update se haya producido de forma correcta.
  
### Manejo de excepciones
- Falta de manejo de errores. No estamos manejando los posibles errores que podríamos encontrar en el código.
Si utilizamos la clase RestControllerAdvice, podríamos tener una clase de manejo de excepciones común, que simplificaría
  la tarea de tratar los errores.
  
### SOLID
- Principio de responsabilidad única clase AdsServiceImpl. Aquí, por un lado estamos 
  listando los anuncios relevantes, los no relevantes y además estamos calculando el score
  de los anuncios. Dejaría en esta clase los métodos #findPublicAds, #findQualityAds y #calculateScores. 
  Sin embargo, el método que calcula el score, que vemos que es muy largo y complejo, lo renombraría 
  a #updateAdScores y sacaría de este servicio el cálculo del score, cumpliendo así con el principio de responsabilidad única.

- Por otro lado, el método AdsServiceImpl#calculateScore tiene bastantes comentarios en el código.
  Eliminaría esos comentarios, puesto que no deberían ser necesarios si el código fuera claro.
  Al mover esta lógica a la clase abstracta (explicado en el siguiente punto), 
  creamos métodos con nombres explicativos.

- InMemoryPersistence#76 puede dar nullPointer si el score viene a null.
  Podemos hacer una comprobación de que el score sea != null.
  ``              
  .filter(x -> x.getScore()!= null && x.getScore() < Constants.FORTY)
  ``
  
- Los métodos AdsServiceImpl.java#findPublicAds y #findQualityAds tienen varios foreach que 
podríamos cambiar por streams.
  
- Los métodos del repositorio AdRepository devuelven una lista de anuncios. En caso de que tuviéramos 
un gran volumen de anuncios, sería interesante trabajar con paginación para mejorar la eficiencia y el 
  rendimiento de la api.


### Patrones de diseño
Para solucionar el problema con el método de #calculateScore(Ad ad) propongo utilizar clases abstractas,
  herencia y el patrón factory method, para crear una factoría de vivienda en la que tuviéramos un método
  que hace un cálculo total de todos los puntos
  ````
  public abstract class Home {

    public int calculateTotalScore(Ad ad){
        int finalScore = Constants.ZERO;

        finalScore += calculateDescriptionLengthScore(ad);
        finalScore += calculateCompleteAdScore(ad);
        finalScore += calculateGarageScore(ad);
        finalScore += calculatePhotosScore(ad);
        finalScore += calculateDescriptionKeyWordsScore(ad);

        return finalScore;
    }
  ````
  en esta clase abstracta, tendríamos los métodos comunes a chalet y piso:
  Home.java#calculatePhotosScore, Home.java#calculateDescriptionKeyWordsScore, Home#calculateGarageScore
  y los métodos abstractos que implementarían las clases Flat y Chalet:
  Home.java#calculateDescriptionLengthScore y Home.java#calculateCompleteAdScore.
  De esta forma, si tenemos que crear nuevos métodos comunes, podríamos implementarlos en la clase Home,
  pero también nos permitiría escalar en el caso de que, por ejemplo, queramos añadir una tipología nueva,
  como local, con un cálculo de puntos particular en ciertos aspectos.
  ````
  public class Chalet extends Home {

    @Override
    public int calculateDescriptionLengthScore(Ad ad){
        // logic to calcula description length score
    }
    @Override
    public int calculateCompleteAdScore(Ad ad){
        // logic to calculate completion score
    }
  }
 
  public class Flat extends Home {

    @Override
    public int calculateDescriptionLengthScore(Ad ad){
        // logic to calcula description length score
    }
    @Override
    public int calculateCompleteAdScore(Ad ad){
        // logic to calculate completion score
    }
  }
  ```` 
  De esta forma, desde el servicio AdsServiceImpl#updateAdScore comprobaríamos
la tipología del anuncio para el cálculo del score:
  ````
  private void updateAdScore(Ad ad) {

        if (Typology.FLAT.equals(ad.getTypology())) {
            home = new Flat();
        }else{
            home = new Chalet();
        }

        int score =  home.calculateTotalScore(ad);
  ````
En este método, podríamos poner un switch para evitar tener que añadir más ifs en el caso
de que tengamos más tipologías.

## Testing
- Faltan tests para la clase controller y también para el repository. Además,
los casos de prueba que se han añadido están poco poblados, y no cubren todos los 
  posibles escenarios del código. También habría que incluir tests de integración.
  
- Podríamos utilizar el SpringRunner para poder hacer uso de algunas anotaciones
propias de spring como @MockBean y @SpyBean, pudiendo inyectar los beans utilizando 
  el contexto de spring.
      