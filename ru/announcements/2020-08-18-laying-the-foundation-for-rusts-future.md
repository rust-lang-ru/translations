---
layout: post
title: Закладывая фундамент будущего Rust
author: The Rust Core Team
release: false
---

Rust был [задуман 2010](https://github.com/rust-lang/rust/commit/c01efc669f09508b55eced32d3c88702578a7c3e) (в зависимости от того, как вы считаете, можно сказать, что в [2006](https://github.com/graydon/rust-prehistory/commit/b0fd440798ab3cfb05c60a1a1bd2894e1618479e)!) как проект в [Mozilla Research](https://research.mozilla.org/), но его долгосрочной целю было создание самостоятельного проекта. В 2015, вместе [с выпуском Rust 1.0](https://blog.rust-lang.org/2015/05/15/Rust-1.0.html), появилось управление независимое от Mozilla. С тех пор, Rust работает как автономная организация, а Mozilla выступает в качестве известного и постоянного финансового и юридического спонсора.

Mozilla была и продолжает восторгаться от возможности широкого использования *и поддержки* языка Rust многими компаниями в отрасли. Сегодня многие компании, как крупные так и малые, используют Rust более разнообразными и значительными способами от [Amazon’s Firecracker](https://aws.amazon.com/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/) до [Fastly’s Lucet](https://www.fastly.com/blog/announcing-lucet-fastly-native-webassembly-compiler-runtime), в критических службах, которые "снабжают энергией" [Discord](https://blog.discord.com/why-discord-is-switching-from-go-to-rust-a190bbca2b1f), [Cloudflare](https://blog.cloudflare.com/enjoy-a-slice-of-quic-and-rust/), [Figma](https://www.figma.com/blog/rust-in-production-at-figma/), [1Password](https://blog.1password.com/1passwordx-december-2019-release/) и многие, многие другие.

On Tuesday, August 11th 2020, Mozilla [announced](https://blog.mozilla.org/blog/2020/08/11/changing-world-changing-mozilla/) their decision to restructure the company and to lay off around 250 people, including folks who are active members of the Rust project and the Rust community. Understandably, these layoffs have generated a lot of uncertainty and confusion about the impact on the Rust project itself. Our goal in this post is to address those concerns. We’ve also got a big announcement to make, so read on!

## Community impact

Нельзя отрицать влияние прошедших увольнений на всех членов сообщества Rust, особенно на людей, потерявших работу в разгар глобальной пандемии. Внезапные, неожиданные увольнения могут быть трудным опытом и они становятся не менее трудными, когда кажется, что мир следит за ними. Уволенных сотрудников, которые ищут помощь в поиске работы, можно найти [в каталоге талантов Mozilla](https://talentdirectory.mozilla.org/).

Несмотря на глубокое влияние, проект Rust в целом очень устойчив к таким событиям. У нас есть лидеры и участники из самых разных слоев общества, работодатели и данное разнообразие является критически важным преимуществом. Более того, распространено заблуждение, что все сотрудники Mozilla, которые участвовали в руководстве разработкой Rust, делали это как часть своей работы. Но фактически, многие сотрудники Mozilla в руководстве Rust [вносили свой вклад в Rust в свое личное время](https://twitter.com/ManishEarth/status/1294023260770770944), а не в рамках своей работы.

Наконец, мы хотели бы подчеркнуть, что участие в командах Rust предоставляется отдельным людям и не связано с работодателем. Сотрудники Mozilla, которые также являются членами команд Rust, продолжают оставаться членами команды сегодня, даже если они попали под увольнение. Конечно, кто-то может отказаться от своего участия. Мы понимаем, что не все могут продолжать вносить свой вклад и полностью поддерживаем их решение. Мы благодарны за все, что они сделали для проекта.

## Starting a foundation

As the project has grown in size, adoption, and maturity, we’ve begun to feel the pains of our success. We’ve developed legal and financial needs that our current organization lacks the capacity to fulfill. While we were able to be successful with Mozilla’s assistance for quite a while, we’ve reached a point where it’s difficult to operate without a legal name, address, and bank account. “How does the Rust project sign a contract?” has become a question we can no longer put off.

В прошлом году мы начали [изучать идею создания независимого фонда Rust](http://smallcultfollowing.com/babysteps/blog/2020/01/09/towards-a-rust-foundation/). Члены команды Rust, имеющие предыдущий опыт работы с фондами для проектов с открытым исходным кодом, собрались вместе, чтобы посмотреть на текущую ситуацию, определить, что нам понадобится от фонда, оценить наши возможности и опросить ключевых членов и директоров из других фондов.

Building on that work, **Mozilla and the Rust Core Team are happy to announce plans to create a Rust foundation. Our goal is to have the first iteration of the foundation up and running by the end of the year.**

This foundation’s first task will be something Rust is already great at: [taking ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html). This time, the resource is legal, rather than something in a program. The various trademarks and domain names associated with Rust, Cargo, and crates.io will move into the foundation, which will also take financial responsibility for the costs they incur. We see this first iteration of the foundation as just the beginning. There’s a lot of possibilities for growing the role of the foundation, and we’re excited to explore those in the future.

For now though, we remain laser-focused on these initial narrow goals for the foundation. As an immediate step the Core Team has [selected members to form a project group](https://www.rust-lang.org/governance/teams/core#project-foundation) driving the efforts to form the foundation. Expect to see follow-up blog posts from the group with more details about the process and opportunities to give feedback. In the meantime, you can email the group at [foundation@rust-lang.org](mailto:foundation@rust-lang.org).

## Развитие инфраструктуры

Хотя мы только начали процесс создания фонда, за последние два года команда по инфраструктуре возглавила работу по снижению зависимости от какой-либо отдельной компании, спонсирующей проект, а также по увеличению числа компаний, поддерживающих Rust.

Данные усилия оказались весьма успешными и как вы можете видеть на нашей [спонсорской странице](https://www.rust-lang.org/sponsors), инфраструктура Rust уже поддерживается рядом различных компаний по всей экосистеме. По мере того, как мы юридически переходим в полностью независимую организацию, команда по инфраструктуре планирует продолжить свои усилия, чтобы гарантировать, что мы не будем чрезмерно полагаться на какого-либо одного спонсора.

## Спасибо

We’re excited to start the next chapter of the project by forming a foundation. We would like to thank everyone we shared this journey with so far: Mozilla for incubating the project and for their support in creating a foundation, our team of leaders and contributors for constantly improving the community and the language, and everyone using Rust for creating the powerful ecosystem that drives so many people to the project. We can’t wait to see what our vibrant community does next.
