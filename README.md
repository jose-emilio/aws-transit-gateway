# **Despliegue de un servicio de salida a Internet centralizado mediante AWS Transit Gateway**

## **Objetivo**

**AWS Transit Gateway** es un servicio administrado que permite gestionar concentradores de tránsito de red que pueden utilizarse para la interconexión de VPCs y redes <em>on-premises</em>.

El objetivo de este repositorio es diseñar una infraestructura de red que permita definir una VPC de Salida llamada `VPC-Servicio`, que se utilizará para encaminar todo el tráfico hacia Internet desde el resto de las VPCs clientes: `VPC-A` y `VPC-B`.

El dispositivo Transit Gateway encaminará las solicitudes hacia Internet a través de la subred privada de la `VPC-Servicio`

Adicionalmente, el tráfico entre la `VPC-A` y `VPC-B` no debe permitirse.

## **Requerimientos**

* Un <em>sandbox</em> de AWS Academy Learner Labs o una cuenta de AWS
* Un entorno Linux con acceso programático configurado a los servicios de AWS

## **Arquitectura propuesta**

* **AWS Transit Gateway** para el despliegue de un concentrador que interconecte selectivamente las diferentes VPCs y encamine el tráfico según las especificaciones del escenario
* **Amazon VPC** para la implementación de las VPCs del escenario
* **NAT Gateway** para proporcionar salida hacia Internet desde la `VPC-Servicio`
* **AWS Systems Manager**, utilizando la característica <em>Session Manager</em> para administrar las instancias EC2 de forma segura

<p align="center">
  <img src="images/aws-transit-gateway.png">
</p>

## **Instrucciones (AWS CloudFormation)**

1. Previamente, se establece la región donde se aprovisionará la infraestructura. En los AWS Academy Learner Labs sólo puede ser `us-east-1` o `us-west-2`:

		REGION=us-east-1

2. Si no se dispone de un bucket de S3 para almacenar los artefactos de AWS CloudFormation, hay que crearlo:

		BUCKET=<nombre-bucket>
		
		aws s3 mb s3://$BUCKET --region $REGION

    Si ya se dispone del bucket de S3, se asigna a la variable de entorno:

        BUCKET=<nombre-bucket>

3. Se empaqueta la plantilla de AWS CloudFormation:

		aws cloudformation package --template-file transit-gateway.yaml --s3-bucket $BUCKET --output-template-file transit-gateway-transformed.yaml --region $REGION

4. Se despliega la infraestructura a partir de la plantilla transformada. El despliegue durará varios minutos:

	Para desplegar en un <em>sandbox</em> de AWS Academy Learner Labs:

		aws cloudformation deploy --template-file transit-gateway-transformed.yaml --stack-name transit-gateway-stack --region $REGION --parameter-overrides AWSAcademy="SI"

	Para desplegar en una cuenta real de AWS:

		aws cloudformation deploy --template-file transit-gateway-transformed.yaml --stack-name transit-gateway-stack --region $REGION --capabilities CAPABILITY_IAM

5. Una vez desplegada la infraestructura, se obtienen las instancias creadas:

        INSTANCIAA=$(aws cloudformation describe-stacks --stack-name transit-gateway-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`InstanciaA`].OutputValue' --output text)

        INSTANCIAB=$(aws cloudformation describe-stacks --stack-name transit-gateway-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`InstanciaB`].OutputValue' --output text)

        INSTANCIASERVICIO=$(aws cloudformation describe-stacks --stack-name transit-gateway-stack --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`InstanciaServicio`].OutputValue' --output text)

    A continuación, se obtienen las IPs privadas de dichas instancias y se anotan aparte:

        IPA=$(aws ec2 describe-instances --filters Name=instance-id,Values=$INSTANCIAA --query Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddress --output text --region $REGION)

        echo $IPA

        IPB=$(aws ec2 describe-instances --filters Name=instance-id,Values=$INSTANCIAB --query Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddress --output text --region $REGION)

        echo $IPB

        IPSERVICIO=$(aws ec2 describe-instances --filters Name=instance-id,Values=$INSTANCIASERVICIO --query Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddress --output text --region $REGION)

        echo $IPSERVICIO

6. A continuación, se lanza la conexión contra la instancia en la `VPC-A` (es necesario tener instalado el plugin de AWS SSM Session Manager para AWS CLI https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-insta
ll-plugin.html):

        aws ssm start-session --target $INSTANCIAA --region $REGION

    Se comprueba la conectividad a Internet, por ejemplo actualizando el repositorio de paquetes mediante la orden:

        sudo yum update -y

    Para comprobar que no existe conectividad con la instancia en la `VPC-B` se ejecuta la siguiente orden, sustituyendo el parámetro por la IP correspondiente:

        ping <IPB>

    Se debe comprobar, además, que sí existe conectividad con la instancia en la `VPC-Servicio`.



