# Contributing

## Contributing to this repo

This repository is managed by EasyCLA. Project participants must sign the free
[GraphQL Specification Membership agreement](https://preview-spec-membership.graphql.org)
before making a contribution. You only need to do this one time, and it can be
signed by
[individual contributors](https://individual-spec-membership.graphql.org) or
their [employers](https://corporate-spec-membership.graphql.org).

To initiate the signature process please open a PR against this repo. The
EasyCLA bot will block the merge if we still need a membership agreement from
you.

You can find
[detailed information here](https://github.com/graphql/graphql-wg/tree/main/membership).
If you have issues, please email
[operations@graphql.org](mailto:operations@graphql.org).

If your company benefits from GraphQL and you would like to provide essential
financial support for the systems and people that power our community, please
also consider membership in the
[GraphQL Foundation](https://foundation.graphql.org/join).

## Running a new build

- Install the latest version of Node and NPM
- Install the Node dependencies with `npm install`
- Build a new spec with `npm run build`
  - This will output the new spec text in `/public` as HTML files

## Contributing to the spec text

### Auto-Formatting

The specification is formatted using the `prettier` tool, so you should not need
to think about gaps between paragraphs and titles, nor about word wrapping -
this is handled for you.

### Adding spec examples

To keep things consistent in the spec, we should use examples that reference an
e-commerce store. We should use types like `Product`, `User`, `Review`, etc.

### Adding new sections

Sections in the spec document are nested under top level sections, then topic
sections, then specific sections. We should aim to limit to 3 nested levels and
consider flattening once we reach 4 levels.
